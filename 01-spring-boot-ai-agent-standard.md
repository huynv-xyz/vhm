# Spring Boot Architecture Standard for AI Agents
# Tài liệu 01: Core standard (một deployable)

Thiết kế repository theo First Principles để AI Agent đọc đúng, hiểu đúng và sửa ít sai nhất, áp dụng được cho cả service nhỏ lẫn hệ thống lớn nhiều team.

| Thuộc tính | Giá trị |
|---|---|
| Phiên bản | 2.2 |
| Trạng thái | Draft, chờ review kiến trúc |
| Phạm vi | Backend Java 21+, Spring Boot 3.x, Gradle (Kotlin DSL). Maven áp dụng tương đương |
| Đối tượng | Tech lead, developer, reviewer, và AI coding agent |
| Owner | `@platform-architecture` (cập nhật theo tổ chức) |

---

## Phần 0. Cách dùng tài liệu này

### 0.0. Bộ tài liệu

| File | Phạm vi | Khi nào đọc |
|---|---|---|
| `01-spring-boot-ai-agent-standard.md` (file này) | Cấu trúc **bên trong một deployable**: monolith, modular monolith, hoặc một service riêng lẻ | Luôn đọc. Là nền cho mọi project |
| `02-spring-boot-ai-agent-microservices.md` | Những gì bổ sung khi hệ thống gồm **nhiều service**: tách service, contract, dữ liệu, saga, resilience, Agent xuyên service | Chỉ khi hệ thống có hoặc sắp có nhiều service |

Tài liệu 02 **không thay thế** tài liệu 01: bên trong mỗi microservice vẫn áp dụng đầy đủ tài liệu này.

### 0.1. Cấu trúc tài liệu

| Phần | Nội dung | Tính chất |
|---|---|---|
| A. Principles | Vì sao thiết kế như vậy | Định hướng, ít thay đổi |
| B. Architecture | Cấu trúc module, layer, dependency | Quy định |
| C. Cross-cutting conventions | Transaction, error, security, event, config, API, DB, observability | Quy định |
| D. Agent context | `AGENTS.md`, feature README, Business Rule ID, ADR | Quy định |
| E. Enforcement | ArchUnit, Spring Modulith, scripts, CI | Executable |
| F. Testing | Chiến lược và quy ước test | Quy định |
| G. Brownfield adoption | Áp dụng cho codebase đang chạy | Lộ trình |
| H. Governance & metrics | Ai duy trì, đo hiệu quả thế nào | Vận hành |
| Phụ lục | Checklist, golden path, template | Tham chiếu |

### 0.2. Từ khóa quy định

Tài liệu dùng từ khóa theo RFC 2119:

- **MUST / MUST NOT**: bắt buộc. Vi phạm phải bị build hoặc review chặn.
- **SHOULD / SHOULD NOT**: mặc định nên làm. Nếu làm khác phải có lý do ghi trong PR hoặc ADR.
- **MAY**: tùy chọn.

### 0.3. Nguyên tắc ưu tiên khi có xung đột

Mọi quy định trong tài liệu này đều có thể bị ghi đè bởi một ADR đã được duyệt trong repository. ADR cụ thể thắng tài liệu chung; tài liệu chung thắng thói quen của team.

---

## Phần A. Principles

### A1. Bài toán gốc

Khi làm việc với source code, một AI Agent phải đi qua chuỗi bước:

```text
Task → Xác định code liên quan → Hiểu business context → Hiểu architecture rule
     → Hiểu code hiện tại → Xác định nơi cần sửa → Sửa code
     → Kiểm tra ảnh hưởng → Chạy verification → Kết luận đúng/sai
```

Bước nào mơ hồ, xác suất sửa sai tăng mạnh. Vì vậy mục tiêu của kiến trúc repository không chỉ là "code sạch", mà là:

> **Giảm tối đa lượng suy luận mà AI Agent phải tự đoán.**

### A2. AI không thật sự "hiểu project"

AI chỉ nhìn thấy tín hiệu: tên file, tên class, package, dependency, comment, tài liệu, test, config và context được cung cấp. Từ đó nó dựng mental model của hệ thống.

Với cấu trúc theo technical layer (`controller/ service/ repository/ util/`), một business rule của Order có thể nằm ở `OrderService`, `OrderHelper`, `OrderMapper`, `OrderEntity`, `OrderValidator` hoặc `CommonUtils`. Agent buộc phải search và đoán.

Repository tốt cho AI làm cho ánh xạ **Intent → Location** gần như deterministic.

### A3. Năm tính chất thiết kế

| Tính chất | Ý nghĩa | Biểu hiện cụ thể trong standard này |
|---|---|---|
| **Locality** | Những gì thuộc cùng một business capability nằm gần nhau | Package by feature (B1) |
| **Explicitness** | Luật được ghi rõ, không để suy luận | `AGENTS.md`, feature README, ADR (Phần D) |
| **Determinism** | Cùng một loại task dẫn đến cùng một nơi | "One obvious place" (B4), naming (B8) |
| **Verifiability** | Luật được máy kiểm tra | ArchUnit, Modulith, `verify.sh` (Phần E) |
| **Context efficiency** | Một task chỉ cần đọc khoảng 3–10 file | Feature README là semantic index, docs gần code |

### A4. Trade-off phải chấp nhận

Standard này không miễn phí. Các chi phí đã biết:

- Tách domain model khỏi JPA entity tốn mapping code và làm mất dirty checking (xem B6 để chọn biến thể).
- Use-case-per-class tạo nhiều file hơn `OrderService` truyền thống.
- Enforcement chặt làm PR đầu tiên chậm hơn.

Đổi lại: Agent và người mới vào team định vị code nhanh hơn, vi phạm kiến trúc bị chặn tự động, rule nghiệp vụ truy vết được. Nếu một project không cần lợi ích này (prototype, script nội bộ, service sống dưới 3 tháng), **không áp dụng standard này**.

### A5. Nguyên tắc tối thượng

> **Thiết kế repository sao cho AI Agent không cần đoán.**

```text
Explicit      > Implicit
Local         > Scattered
Specific      > Generic
Executable    > Documentation-only
Business meaning > Technical ceremony
Boring + obvious > Clever + abstract
```

Kiến trúc tốt cho AI **không** phải kiến trúc nhiều layer nhất. Nó là kiến trúc mà từ một task, Agent trả lời nhanh được năm câu hỏi:

1. Tôi phải đọc gì?
2. Tôi phải sửa ở đâu?
3. Rule nào đang áp dụng?
4. Điều gì không được phép?
5. Làm sao biết thay đổi của tôi đúng?

---

## Phần B. Architecture

### B1. Package by feature, enforce bằng module boundary

Project **MUST** tổ chức theo business capability, không theo technical layer.

```text
com.company.app
├── Application.java
├── order/
├── customer/
├── payment/
└── common/
```

Tuy nhiên package-by-feature trong một module Gradle duy nhất **không tự enforce boundary**: mọi class `public` vẫn import được từ mọi nơi. Vì vậy chọn cơ chế enforce theo quy mô:

| Quy mô | Đặc điểm | Cơ chế enforce |
|---|---|---|
| Nhỏ | 1 team, dưới ~5 feature | Single module + ArchUnit (E1) |
| Vừa và lớn (mặc định khuyến nghị) | Nhiều feature, 1–3 team, một deployable | **Spring Modulith** + ArchUnit (E2) |
| Rất lớn | Nhiều team, cần compile-time isolation, build time quan trọng | Gradle multi-module theo feature, hoặc tách service (xem tài liệu 02) |

Quyết định này **MUST** được ghi trong `ADR-001-module-strategy.md`.

### B2. Cấu trúc repository

```text
my-service/
├── AGENTS.md                      # Operating manual cho Agent (D1)
├── README.md                      # Cho người: chạy, build, deploy
├── CODEOWNERS
├── build.gradle.kts
├── settings.gradle.kts
├── gradle/libs.versions.toml      # Version catalog, một nguồn version duy nhất
│
├── docs/
│   ├── architecture/
│   │   ├── overview.md            # System context, module map
│   │   └── modules/               # Sinh tự động bởi Spring Modulith Documenter
│   ├── business/
│   │   └── glossary.md            # Ubiquitous language
│   ├── conventions/
│   │   ├── api.md
│   │   ├── database.md
│   │   ├── errors.md
│   │   └── events.md
│   └── adr/
│       ├── README.md              # Index ADR + trạng thái
│       └── ADR-NNN-*.md
│
├── src/main/java/com/company/app/
│   ├── Application.java
│   ├── common/                    # Shared kernel, kiểm soát chặt (B9)
│   ├── customer/
│   ├── order/
│   └── payment/
│
├── src/main/resources/
│   ├── application.yml
│   ├── application-local.yml
│   ├── db/migration/
│   └── openapi/openapi.yaml
│
├── src/test/java/com/company/app/
│   ├── architecture/              # ArchUnit + Modulith tests
│   ├── customer/
│   ├── order/
│   └── payment/
├── src/test/resources/archunit_store/   # Frozen violations (brownfield)
│
├── scripts/
│   ├── verify.sh                  # Lệnh verification duy nhất
│   ├── check-business-rules.sh
│   └── run-local.sh
│
└── .github/
    ├── workflows/
    └── pull_request_template.md
```

### B3. Cấu trúc một feature

```text
order/
├── README.md                      # Semantic index (D2)
├── OrderQueries.java              # PUBLIC API của module cho module khác (B5)
├── OrderSummary.java              # DTO thuộc public API
│
├── api/                           # INBOUND adapters: mọi thứ gọi VÀO feature
│   ├── http/
│   │   ├── OrderController.java
│   │   ├── CreateOrderRequest.java
│   │   └── OrderResponse.java
│   ├── messaging/
│   │   └── PaymentCompletedListener.java
│   └── job/
│       └── ExpireDraftOrdersJob.java
│
├── application/                   # Use cases: điều phối, transaction, authorization
│   ├── CreateOrderUseCase.java
│   ├── CreateOrderCommand.java
│   ├── CancelOrderUseCase.java
│   ├── ConfirmOrderUseCase.java
│   └── query/
│       ├── GetOrderDetailsQuery.java
│       ├── OrderDetails.java
│       └── OrderReadModel.java    # Port cho read side, implement ở infrastructure
│
├── domain/                        # Business truth: state, behavior, invariant
│   ├── Order.java
│   ├── OrderId.java
│   ├── OrderItem.java
│   ├── OrderStatus.java
│   ├── CustomerSnapshot.java
│   ├── OrderRepository.java       # Port
│   ├── CustomerDirectory.java     # Port tới module khác
│   ├── event/
│   │   ├── OrderCreated.java
│   │   └── OrderCancelled.java
│   └── exception/
│
└── infrastructure/                # OUTBOUND adapters: feature gọi RA thế giới bên ngoài
    ├── persistence/
    │   ├── OrderEntity.java
    │   ├── OrderJpaRepository.java
    │   ├── OrderPersistenceMapper.java
    │   ├── JpaOrderRepository.java       # implements domain.OrderRepository
    │   └── JpaOrderReadModel.java        # implements application.query.OrderReadModel
    ├── customer/
    │   └── CustomerDirectoryAdapter.java # gọi customer.CustomerQueries
    └── OrderQueriesImpl.java             # implements public API OrderQueries
```

Điểm khác biệt quan trọng so với cách chia "infrastructure chứa mọi thứ kỹ thuật":

- **`api/` = inbound**: HTTP controller, Kafka/Rabbit listener, scheduled job, CLI. Tất cả đều là "thế giới bên ngoài gọi vào", nên tất cả đều được gọi use case.
- **`infrastructure/` = outbound**: database, HTTP client, message publisher, cache, storage. Chỉ implement port, **không** gọi use case.

Cách chia này loại bỏ câu hỏi "Kafka consumer nằm đâu và nó được gọi ai".

### B4. Trách nhiệm từng layer

| Layer | Trả lời câu hỏi | MUST chứa | MUST NOT chứa |
|---|---|---|---|
| `api/` | Bên ngoài giao tiếp với feature thế nào? | Validate format input, map request → command, gọi use case, map result → response | Business decision, query DB, gọi repository, `@Transactional` |
| `application/` | Một use case được thực hiện thế nào? | Load → gọi domain → persist → publish event; transaction boundary; authorization | Business invariant cốt lõi, SQL, chi tiết HTTP |
| `domain/` | Business đúng/sai theo luật nào? | Aggregate, value object, invariant, domain event, port interface | Spring, JPA (xem B6), Kafka, HTTP, Jackson |
| `infrastructure/` | Port được nối với công nghệ thật thế nào? | JPA, SQL, HTTP client, publisher, cache | Business rule, gọi use case |

Quy tắc "one obvious place":

| Câu hỏi | Câu trả lời duy nhất |
|---|---|
| Business rule của Order nằm đâu? | `order/domain` |
| Endpoint HTTP của Order nằm đâu? | `order/api/http` |
| Ai lắng nghe event `PaymentCompleted`? | `order/api/messaging` |
| Transaction mở ở đâu? | Use case trong `order/application` |
| Kiểm tra quyền ở đâu? | Use case trong `order/application` |
| JPA mapping nằm đâu? | `order/infrastructure/persistence` |
| Module khác lấy thông tin Order qua đâu? | `order/OrderQueries.java` (root package) |

### B5. Dependency rules

#### Trong một feature

```text
            api (inbound)
                │
                ▼
           application ◄──────────┐
                │                 │ (chỉ implement port của application.query)
                ▼                 │
             domain ◄──────── infrastructure (outbound)
```

| Từ \ Đến | api | application | domain | infrastructure |
|---|---|---|---|---|
| **api** | ✓ | ✓ | Chỉ value type (ID, enum, exception). MUST NOT gọi repository hoặc method thay đổi state | ✗ |
| **application** | ✗ | ✓ | ✓ | ✗ |
| **domain** | ✗ | ✗ | ✓ | ✗ |
| **infrastructure** | ✗ | Chỉ để implement interface trong `application/query`. MUST NOT gọi use case | ✓ | ✓ |

#### Giữa các feature

Một feature **MUST NOT** import package con (`api`, `application`, `domain`, `infrastructure`) của feature khác. Giao tiếp chỉ qua ba kênh, theo thứ tự ưu tiên:

1. **Domain event** (bất đồng bộ, coupling thấp nhất). Dùng khi feature kia chỉ cần *phản ứng* với điều đã xảy ra.
2. **Public API của module** (đồng bộ). Là các type nằm ở **root package** của feature, ví dụ `customer.CustomerQueries`. Đây đúng là quy ước của Spring Modulith: root package là API, package con là internal.
3. **Shared kernel** trong `common/` cho value type thật sự dùng chung (`Money`, `TenantId`).

Ví dụ BR-ORDER-001 "chỉ customer ACTIVE được tạo order". Rule thuộc Order nhưng dữ liệu thuộc Customer:

```text
order/domain/CustomerDirectory          (port, do Order định nghĩa theo ngôn ngữ của Order)
order/domain/CustomerSnapshot           (value object: customerId, active)
order/domain/Order.create(..., snapshot)   ← rule nằm ở đây
order/infrastructure/customer/CustomerDirectoryAdapter
        └── gọi customer.CustomerQueries  (public API của Customer)
```

Order không biết `CustomerEntity`, không biết bảng `customers`, không biết Customer lưu trạng thái thế nào. Nếu sau này Customer tách thành service riêng, chỉ `CustomerDirectoryAdapter` phải đổi.

### B6. Domain model và JPA entity

Đây là quyết định có chi phí thật, nên standard cho phép hai biến thể. Repository **MUST** chọn một và ghi trong `ADR-003-persistence-model.md`.

| | Biến thể A: Tách riêng (mặc định) | Biến thể B: Domain là JPA entity |
|---|---|---|
| Domain | Pure Java | Có annotation `jakarta.persistence` |
| Ưu điểm | Domain test không cần JPA; đổi persistence không ảnh hưởng domain; Agent thấy rule mà không bị nhiễu bởi mapping | Ít code; giữ dirty checking, lazy loading |
| Nhược điểm | Mapping code; phải tự xử lý version và reconstitution | Domain dính framework; dễ rò rỉ lazy-loading vào business logic |
| Phù hợp | Domain phức tạp, sống lâu, nhiều rule | CRUD-heavy, domain mỏng |

Với biến thể A, domain **MUST** có hai đường tạo object riêng biệt và **MUST** mang theo version để giữ optimistic locking:

```java
public final class Order {

    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderItem> items;
    private final long version;
    private OrderStatus status;
    private final List<Object> domainEvents = new ArrayList<>();

    private Order(OrderId id, CustomerId customerId, List<OrderItem> items,
                  OrderStatus status, long version) {
        this.id = id;
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
        this.status = status;
        this.version = version;
    }

    /** Tạo mới: áp dụng toàn bộ invariant khởi tạo. */
    public static Order create(OrderId id, CustomerSnapshot customer, List<OrderItem> items) {
        // BR-ORDER-001
        if (!customer.active()) {
            throw new CustomerNotActiveException(customer.customerId());
        }
        // BR-ORDER-003
        if (items.isEmpty()) {
            throw new EmptyOrderException();
        }
        Order order = new Order(id, customer.customerId(), items, OrderStatus.DRAFT, 0);
        order.domainEvents.add(new OrderCreated(id, customer.customerId()));
        return order;
    }

    /** Tái tạo từ persistence: KHÔNG chạy lại invariant khởi tạo, KHÔNG phát event. */
    public static Order reconstitute(OrderId id, CustomerId customerId, List<OrderItem> items,
                                     OrderStatus status, long version) {
        return new Order(id, customerId, items, status, version);
    }

    public void confirm() {
        requireStatus(OrderStatus.DRAFT);
        status = OrderStatus.CONFIRMED;
    }

    public void cancel() {
        // BR-ORDER-004
        if (status == OrderStatus.COMPLETED || status == OrderStatus.CANCELLED) {
            throw new InvalidOrderStateException(id, status, "cancel");
        }
        status = OrderStatus.CANCELLED;
        domainEvents.add(new OrderCancelled(id));
    }

    public List<Object> pullDomainEvents() {
        List<Object> events = List.copyOf(domainEvents);
        domainEvents.clear();
        return events;
    }

    private void requireStatus(OrderStatus expected) {
        if (status != expected) {
            throw new InvalidOrderStateException(id, status, "requires " + expected);
        }
    }

    // getters: id(), customerId(), items() (trả bản copy), status(), version()
}
```

`JpaOrderRepository.save()` map domain sang `OrderEntity` **kèm `version`**, nên `@Version` trên entity vẫn phát hiện được concurrent update.

### B7. Use case

Mỗi use case là một class, tên là intent. Không tạo `OrderService` 40 method.

```java
@Service
@RequiredArgsConstructor
public class CreateOrderUseCase {

    private final OrderRepository orders;
    private final CustomerDirectory customers;
    private final ApplicationEventPublisher events;
    private final OrderAuthorization authorization;

    @Transactional
    public OrderId execute(CreateOrderCommand command) {
        authorization.requireCanCreateFor(command.customerId());

        CustomerSnapshot customer = customers.find(command.customerId())
                .orElseThrow(() -> new CustomerNotFoundException(command.customerId()));

        Order order = Order.create(orders.nextId(), customer, command.items());

        orders.save(order);
        order.pullDomainEvents().forEach(events::publishEvent);
        return order.id();
    }
}
```

Use case chịu trách nhiệm: authorization → load → gọi domain → persist → publish event. Rule nghiệp vụ nằm trong `Order.create`, không nằm trong use case.

**Read side**: query **SHOULD NOT** load aggregate qua `OrderRepository` chỉ để trả dữ liệu. Dùng port `application/query/OrderReadModel` trả projection, implement bằng JPQL projection, jOOQ hoặc SQL ở `infrastructure/persistence`. Read side không cần đi qua domain vì nó không thay đổi state.

### B8. Naming

Tên class **MUST** thể hiện intent và **SHOULD** search-friendly.

| Tránh | Dùng |
|---|---|
| `OrderService`, `OrderManager`, `OrderProcessor`, `OrderHandler` | `CreateOrderUseCase`, `CancelOrderUseCase`, `OrderCancellationPolicy` |
| `OrderHelper`, `CommonUtils`, `AppHelper` | Đặt theo trách nhiệm: `money/Money`, `time/BusinessCalendar` |
| `PaymentService` + `PaymentServiceImpl` (1 implementation) | Class cụ thể không interface; hoặc `PaymentGateway` + `StripePaymentGateway`, `MomoPaymentGateway` khi có boundary thật |
| `IOrderRepository` | `OrderRepository` |

Hậu tố chuẩn:

| Hậu tố | Layer | Ý nghĩa |
|---|---|---|
| `*UseCase` | application | Use case thay đổi state |
| `*Query` | application/query | Use case chỉ đọc |
| `*Command` | application | Input của use case |
| `*Controller`, `*Listener`, `*Job` | api | Inbound adapter |
| `*Request`, `*Response` | api/http | DTO transport |
| `*Policy` | domain | Rule nghiệp vụ tách riêng khỏi aggregate |
| `*Repository` | domain | Port persistence của aggregate |
| `Jpa*`, `Kafka*`, `Http*` | infrastructure | Adapter theo công nghệ |

Nguyên tắc abstraction: **mỗi interface phải loại bỏ một coupling hoặc mô tả một boundary thật**. Port trong domain (`OrderRepository`, `CustomerDirectory`) là boundary thật. `OrderServiceImpl` thì không.

### B9. `common/` là shared kernel, không phải thùng rác

Một class **MAY** vào `common/` chỉ khi thỏa cả ba điều kiện:

1. Không thuộc business feature cụ thể nào.
2. Được ít nhất hai feature dùng **ngay bây giờ** (không phải "sau này có thể").
3. Có tên mang nghĩa rõ ràng.

```text
common/
├── domain/          # Money, TenantId, DomainException (base)
├── web/             # GlobalExceptionHandler, ProblemDetail mapping
├── security/        # Authentication, CurrentUser
└── observability/   # Logging, tracing config
```

**MUST NOT** có: `common/util/`, `common/helper/`, `common/order/`, `Constants.java` dùng chung toàn hệ thống.

Thay đổi trong `common/` ảnh hưởng mọi feature, nên `common/` **MUST** có owner riêng trong `CODEOWNERS`.

---

## Phần C. Cross-cutting conventions

Mỗi mục dưới đây là "hidden knowledge" thường chỉ tồn tại trong đầu senior. Với Agent, những gì không được viết ra thì không tồn tại.

### C1. Transaction

- `@Transactional` **MUST** chỉ xuất hiện trong `application/` (enforce ở E1).
- Một use case **SHOULD** tương ứng một transaction.
- Use case **MUST NOT** gọi HTTP bên ngoài bên trong transaction. Nếu cần, gọi trước transaction hoặc dùng event sau commit.
- Read-only query **SHOULD** dùng `@Transactional(readOnly = true)`.
- Phản ứng với event sau commit dùng `@TransactionalEventListener(phase = AFTER_COMMIT)` hoặc `@ApplicationModuleListener` (Spring Modulith).

### C2. Error handling

Tất cả lỗi HTTP **MUST** trả về `ProblemDetail` theo RFC 9457 với một `code` ổn định mà client có thể dựa vào.

```java
// common/domain
public abstract class DomainException extends RuntimeException {
    protected DomainException(String message) { super(message); }
    public abstract String code();      // ví dụ: "ORDER_INVALID_STATE"
}
```

| Loại exception | HTTP status | Ví dụ code |
|---|---|---|
| `*NotFoundException` | 404 | `ORDER_NOT_FOUND` |
| Vi phạm invariant (`InvalidOrderStateException`) | 409 | `ORDER_INVALID_STATE` |
| Vi phạm rule nghiệp vụ đầu vào (`CustomerNotActiveException`) | 422 | `ORDER_CUSTOMER_NOT_ACTIVE` |
| Validation format (`MethodArgumentNotValidException`) | 400 | `VALIDATION_FAILED` |
| Không có quyền | 403 | `FORBIDDEN` |
| Lỗi không lường trước | 500 | `INTERNAL_ERROR` (không lộ stack trace) |

Mapping nằm ở **một nơi duy nhất**: `common/web/GlobalExceptionHandler`. Controller **MUST NOT** tự try/catch để tạo response lỗi. Danh sách code đầy đủ nằm trong `docs/conventions/errors.md`; code đã public **MUST NOT** bị đổi tên.

### C3. Security

- **Authentication** (bạn là ai) nằm ở `common/security`, cấu hình bằng Spring Security filter chain.
- **Authorization** (bạn được làm gì) **MUST** nằm trong use case ở `application/`, không ở controller.

Lý do: cùng một use case có thể được gọi từ HTTP, message listener hoặc job. Nếu kiểm tra quyền ở controller, các đường vào khác sẽ bỏ qua nó.

- Với multi-tenant: mọi query **MUST** lọc theo tenant. Cơ chế (Hibernate filter, row-level security, tham số tường minh) ghi trong ADR.
- Secret **MUST NOT** nằm trong repository, kể cả `application-local.yml`.

### C4. Domain event và integration event

| | Domain event | Integration event |
|---|---|---|
| Phạm vi | Trong cùng deployable | Ra ngoài service (Kafka, RabbitMQ) |
| Vị trí | `feature/domain/event/` | Schema trong `docs/conventions/events.md` hoặc schema registry |
| Đặt tên | Quá khứ: `OrderCreated`, `OrderCancelled` | `order.created.v1` |
| Thay đổi | Tự do refactor | Có version, MUST tương thích ngược |

Quy tắc:

- Publish integration event **MUST** đi qua outbox (Spring Modulith Event Publication Registry, hoặc bảng outbox tự quản lý) để không mất event khi commit DB thành công nhưng broker lỗi.
- Consumer **MUST** idempotent: lưu `eventId` đã xử lý hoặc thiết kế thao tác idempotent tự nhiên.
- Retry và dead-letter policy ghi trong `docs/conventions/events.md`.

### C5. Configuration

Spring Boot áp dụng thứ tự ưu tiên (sau ghi đè trước):

```text
application.yml (trong jar)
  ↓ bị ghi đè bởi
application-{profile}.yml
  ↓ bị ghi đè bởi
Environment variables (bao gồm ConfigMap mount dạng env)
  ↓ bị ghi đè bởi
Command-line arguments

Secret: lấy qua spring.config.import (Vault, AWS Secrets Manager...) hoặc Secret mount
```

Quy định:

- Profile file **SHOULD** chỉ có `local` và `test`. Khác biệt giữa sit/uat/prod **SHOULD** đi qua environment variable, không qua `application-prod.yml`. Điều này tránh tình trạng Agent sửa `application-prod.yml` mà không biết production thực ra đọc từ ConfigMap.
- Mọi property tự định nghĩa **MUST** bind qua `@ConfigurationProperties` có validation, không dùng `@Value` rải rác.
- `docs/conventions/configuration.md` **MUST** liệt kê property bắt buộc theo môi trường và nơi lưu secret.

### C6. API contract

- Repository **MUST** chọn một trong hai: **contract-first** (`openapi.yaml` là nguồn gốc, sinh interface) hoặc **code-first** (sinh `openapi.yaml` từ code và commit lại). Ghi trong ADR.
- Path **MUST** có version: `/v1/orders`.
- Breaking change (xóa field, đổi kiểu, đổi status code, thêm field bắt buộc ở request) **MUST** tạo version mới hoặc được duyệt rõ ràng.
- CI **SHOULD** chạy công cụ diff OpenAPI để phát hiện breaking change tự động.

### C7. Database migration

- Schema **MUST** thay đổi qua Flyway (hoặc Liquibase), không qua `ddl-auto`. `spring.jpa.hibernate.ddl-auto` **MUST** là `validate` hoặc `none` ngoài môi trường test.
- Tên file **SHOULD** dùng timestamp để tránh trùng giữa nhiều team: `V20261004_1530__add_order_cancel_reason.sql`.
- Migration đã merge vào nhánh chính **MUST NOT** bị sửa. Muốn đổi thì tạo migration mới.
- Thay đổi không downtime **MUST** theo expand → migrate → contract: thêm cột mới, deploy code ghi cả hai, backfill, chuyển đọc, rồi mới xóa cột cũ ở release sau.
- Một thay đổi persistence hoàn chỉnh gồm: entity + migration + test repository. Thiếu một trong ba là chưa xong.

### C8. Observability

- Log **SHOULD** dạng structured (Spring Boot 3.4+ hỗ trợ sẵn qua `logging.structured.format.console`).
- Mọi request và message **MUST** mang trace id (Micrometer Tracing).
- Log **MUST NOT** chứa PII, token, mật khẩu, số thẻ.
- Log nghiệp vụ quan trọng (đổi trạng thái order, thanh toán) **SHOULD** ở use case, kèm id của aggregate.

---

## Phần D. Agent context

### D1. `AGENTS.md`

`AGENTS.md` là operating manual cho Agent, đặt ở root. Các nguyên tắc:

- **Ngắn**: root `AGENTS.md` **SHOULD** dưới khoảng 150 dòng. Nó được nạp vào context của mọi task, nên mỗi dòng đều có chi phí. Chi tiết đặt ở `docs/` và link tới.
- **Phân tầng**: feature có quy tắc riêng **MAY** có `AGENTS.md` trong thư mục feature. Agent áp dụng file gần nhất với file đang sửa; file con chỉ ghi phần khác biệt, không lặp lại root.
- **Đúng sự thật**: `AGENTS.md` mô tả trạng thái *hiện tại* của repo. Với codebase legacy, ghi rõ phần nào chưa tuân thủ (xem Phần G), đừng mô tả kiến trúc lý tưởng không tồn tại.
- **Tương thích công cụ**: với công cụ đọc tên file khác (ví dụ `CLAUDE.md`), tạo file đó chỉ để trỏ về `AGENTS.md`, không duy trì hai bản nội dung.

Template:

```md
# Agent Instructions

## Project
Order management service. Java 21, Spring Boot 3.x, Gradle, PostgreSQL, Kafka.
Module strategy: Spring Modulith (see docs/adr/ADR-001-module-strategy.md).

## Before you change anything
1. Identify the affected feature (top-level package under com.company.app).
2. Read <feature>/README.md. Find the business rule IDs (BR-*) related to the task.
3. Read the relevant use case in <feature>/application and the aggregate in <feature>/domain.
4. Read existing tests for those rules: grep the BR ID in src/test.
5. If the task touches architecture, read the related ADR in docs/adr.

## Where things go
- Business rules: <feature>/domain. Never in controllers or use cases.
- Use cases, transactions, authorization: <feature>/application.
- Inbound (HTTP, listeners, jobs): <feature>/api.
- Outbound (DB, HTTP clients, publishers): <feature>/infrastructure.
- Other features: only via their root-package API or domain events.
Full rules: docs/architecture/overview.md

## Hard rules (build will fail or PR will be rejected)
- Domain must not depend on Spring, JPA, Kafka or HTTP libraries.
- @Transactional only in application layer.
- No *Utils, *Helper, *Manager, *ServiceImpl classes.
- Never import another feature's api/application/domain/infrastructure packages.
- Every new or changed business rule needs a BR ID in the feature README and a test tagged with it.
- Schema changes need a new Flyway migration. Never edit an existing migration.

## You MUST NOT
- Edit files in src/main/resources/db/migration that already exist.
- Edit src/test/resources/archunit_store/ or weaken tests in src/test/java/**/architecture.
- Add @Disabled, delete failing tests, or loosen assertions to make the build pass.
- Change openapi.yaml in a backward-incompatible way.
- Add new dependencies to gradle/libs.versions.toml without asking.
- Touch secrets, CI credentials, or deployment config.

## Stop and ask a human when
- README, code and tests disagree about a business rule.
- The task requires a rule that is not documented anywhere.
- The change needs to modify the domain of more than one feature.
- The change needs a new architectural pattern (requires an ADR).
- ./scripts/verify.sh fails for reasons unrelated to your change.

## Verify
- Fast loop while working: ./scripts/verify.sh --fast
- Before finishing: ./scripts/verify.sh   (must pass)

## When you finish, report
- What changed and why (one paragraph).
- Business rules affected (BR IDs).
- Files changed, grouped by feature and layer.
- Docs updated (README, ADR) or why none were needed.
- Anything you were unsure about.

## Source of truth
See docs/architecture/overview.md#source-of-truth. On conflict: stop and ask.
```

### D2. Feature README

Feature README là **semantic index** của module: Agent đọc nó trước khi mở code. Mỗi feature **MUST** có README theo template:

```md
# Order

## Responsibility
Manages the lifecycle of customer orders from draft to completion.
Does NOT handle payment capture (payment module) or customer data (customer module).

## Owner
@team-checkout

## Main concepts
- Order (aggregate root): domain/Order.java
- OrderItem: domain/OrderItem.java
- OrderStatus: domain/OrderStatus.java

## Lifecycle
DRAFT → CONFIRMED → PROCESSING → COMPLETED
DRAFT | CONFIRMED | PROCESSING → CANCELLED

## Business rules
| ID | Rule | Enforced in |
|---|---|---|
| BR-ORDER-001 | Only ACTIVE customers may create orders | Order.create |
| BR-ORDER-002 | A CONFIRMED order cannot be modified | Order.changeItems |
| BR-ORDER-003 | An order must contain at least one item | Order.create |
| BR-ORDER-004 | COMPLETED or CANCELLED orders cannot be cancelled | Order.cancel |

## Use cases
| Use case | Entry points |
|---|---|
| CreateOrderUseCase | POST /v1/orders |
| CancelOrderUseCase | POST /v1/orders/{id}/cancel |
| ConfirmOrderUseCase | PaymentCompletedListener (payment.completed.v1) |
| GetOrderDetailsQuery | GET /v1/orders/{id} |

## Public API (for other modules)
- OrderQueries: findSummary(OrderId)

## Events
Published: OrderCreated, OrderCancelled (domain); order.created.v1 (integration)
Consumed: payment.completed.v1

## Dependencies
- customer module: via CustomerDirectory → customer.CustomerQueries
- Tables: orders, order_items

## Known quirks
- Legacy orders before 2024 may have null customer_id. Do not "fix" with a migration.
```

Mục **Known quirks** là nơi ghi lại hidden knowledge kiểu "cái này phải hỏi anh A". Nó thường là mục giá trị nhất với Agent.

### D3. Business Rule ID

Business rule quan trọng **MUST** có ID ổn định dạng `BR-<FEATURE>-<NNN>`, xuất hiện ở ba nơi:

| Nơi | Hình thức |
|---|---|
| Feature README | Dòng trong bảng Business rules |
| Code | Comment `// BR-ORDER-001` ngay trên dòng enforce rule |
| Test | `@Tag("BR-ORDER-001")` trên test method |

```java
@Test
@Tag("BR-ORDER-001")
void inactive_customer_cannot_create_order() {
    var customer = CustomerSnapshots.inactive();

    assertThatThrownBy(() -> Order.create(anOrderId(), customer, List.of(anItem())))
            .isInstanceOf(CustomerNotActiveException.class);
}
```

Dùng `@Tag` thay vì nhét ID vào tên method vì: tên method giữ đúng convention, và có thể chạy riêng test của một rule bằng `--tests`/tag filter.

Quy tắc vòng đời:

- ID **MUST NOT** được tái sử dụng. Rule bị bỏ thì đánh dấu `(retired)` trong README, không xóa dòng.
- Đổi nội dung rule thì giữ ID, cập nhật mô tả và test trong cùng PR.
- `scripts/check-business-rules.sh` (E4) kiểm tra tự động trong CI: rule có trong README phải có test, ID có trong code/test phải có trong README.

Agent chỉ cần `grep -r BR-ORDER-001` là thấy toàn bộ vòng đời của rule: tài liệu, code, test.

### D4. ADR

Quyết định kiến trúc **MUST NOT** tồn tại dưới dạng "team đều biết". Mỗi ADR một file trong `docs/adr/`, có index trong `docs/adr/README.md`.

ADR tối thiểu cho repository theo standard này:

| ADR | Nội dung |
|---|---|
| ADR-001-module-strategy | Single module / Spring Modulith / multi-module |
| ADR-002-transaction-boundary | Transaction ở use case |
| ADR-003-persistence-model | Biến thể A hay B (B6) |
| ADR-004-integration-events | Outbox, broker, schema versioning |
| ADR-005-api-contract | Contract-first hay code-first |
| ADR-006-authorization | Cơ chế kiểm tra quyền, multi-tenancy |

Template:

```md
# ADR-002: Transaction boundary

Status: Accepted          <!-- Proposed | Accepted | Superseded by ADR-NNN | Deprecated -->
Date: 2026-10-04
Deciders: @platform-architecture

## Context
We need predictable transaction ownership that an agent can locate by reading one class.

## Decision
Application use cases own transaction boundaries.

## Rules
- @Transactional is allowed only in the application layer.
- Domain objects never manage transactions.
- No outbound HTTP calls inside a transaction.

## Enforcement
ArchitectureTest.transactional_only_in_application

## Consequences
+ Transaction scope is visible by reading one use case.
- Cross-use-case composition needs events or a new use case, not nested calls.

## Alternatives considered
Transactions in controllers: rejected, mixes transport and consistency concerns.
```

Mục **Enforcement** bắt buộc khi rule có thể kiểm tra bằng máy. ADR không có enforcement chỉ là lời khuyên.

### D5. Source of truth

Khi các nguồn mâu thuẫn, thứ tự tin cậy là:

```text
1. ADR đã Accepted         (quyết định có chủ đích)
2. Feature README: Business rules   (ý định nghiệp vụ đã được xác nhận)
3. Tests                   (hành vi được kỳ vọng)
4. Code                    (hành vi thực tế)
5. Tài liệu chung khác
```

Business rule đứng trên test vì test có thể encode chính bug. Code đứng thấp nhất vì nó là thứ đang được sửa.

Tuy nhiên thứ tự này chỉ để **hiểu**, không phải để **tự quyết**. Quy trình bắt buộc khi phát hiện mâu thuẫn:

1. Agent **MUST NOT** tự chọn một bên và sửa các bên còn lại cho khớp.
2. Agent dừng lại, báo cáo: nguồn nào nói gì, ở file nào, dòng nào.
3. Con người quyết định, rồi cập nhật tất cả các nguồn trong cùng một PR.

### D6. Tài liệu gần code

| Loại thông tin | Nơi đặt |
|---|---|
| Kiến trúc toàn hệ thống, convention | `docs/` |
| Ngữ nghĩa và rule của một feature | `<feature>/README.md` |
| Quyết định kiến trúc | `docs/adr/` |
| Hành vi chi tiết | Domain code + test |
| Module map, dependency giữa module | Sinh tự động (Spring Modulith `Documenter`), không viết tay |

**MUST NOT** gom mọi thứ vào một `docs/system.md` hàng nghìn dòng. Thông tin có thể sinh tự động thì **SHOULD** sinh tự động, vì tài liệu viết tay luôn có nguy cơ lỗi thời.

---

## Phần E. Enforcement

Documentation là soft constraint. Mục tiêu của phần này là biến "Agent không nên làm sai" thành "Agent làm sai thì build fail". Mọi rule MUST trong Phần B và C có thể kiểm tra bằng máy đều **MUST** có test tương ứng ở đây.

### E1. ArchUnit: rule trong feature và rule toàn cục

```java
package com.company.app.architecture;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;
import org.springframework.transaction.annotation.Transactional;

import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.classes;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noClasses;
import static com.tngtech.archunit.lang.syntax.ArchRuleDefinition.noMethods;

@AnalyzeClasses(packages = "com.company.app", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    // ---------- Domain purity (B4, B6) ----------

    @ArchTest
    static final ArchRule domain_is_framework_free = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "org.springframework..",
                    "jakarta.persistence..",   // bỏ dòng này nếu ADR-003 chọn biến thể B
                    "org.hibernate..",
                    "org.apache.kafka..",
                    "com.fasterxml.jackson..",
                    "jakarta.servlet..")
            .because("ADR-003: domain is plain Java");

    @ArchTest
    static final ArchRule domain_depends_on_nothing_outward = noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage(
                    "..application..", "..api..", "..infrastructure..");

    // ---------- Layer direction (B5) ----------

    @ArchTest
    static final ArchRule application_does_not_know_adapters = noClasses()
            .that().resideInAPackage("..application..")
            .should().dependOnClassesThat().resideInAnyPackage("..api..", "..infrastructure..");

    @ArchTest
    static final ArchRule api_does_not_touch_infrastructure = noClasses()
            .that().resideInAPackage("..api..")
            .should().dependOnClassesThat().resideInAPackage("..infrastructure..");

    @ArchTest
    static final ArchRule api_does_not_use_repositories = noClasses()
            .that().resideInAPackage("..api..")
            .should().dependOnClassesThat().haveSimpleNameEndingWith("Repository")
            .because("controllers and listeners go through use cases");

    @ArchTest
    static final ArchRule infrastructure_does_not_call_use_cases = noClasses()
            .that().resideInAPackage("..infrastructure..")
            .should().dependOnClassesThat().haveSimpleNameEndingWith("UseCase")
            .orShould().dependOnClassesThat().resideInAPackage("..api..");

    // ---------- Transaction (C1) ----------

    @ArchTest
    static final ArchRule transactional_classes_only_in_application = noClasses()
            .that().resideOutsideOfPackage("..application..")
            .should().beAnnotatedWith(Transactional.class)
            .because("ADR-002: use cases own transactions");

    @ArchTest
    static final ArchRule transactional_methods_only_in_application = noMethods()
            .that().areDeclaredInClassesThat().resideOutsideOfPackage("..application..")
            .should().beAnnotatedWith(Transactional.class)
            .because("ADR-002: use cases own transactions");

    // ---------- Naming (B8, B9) ----------

    @ArchTest
    static final ArchRule no_generic_names = noClasses()
            .should().haveNameMatching(".*(Utils|Util|Helper|Manager|ServiceImpl)$")
            .because("names must express responsibility");

    @ArchTest
    static final ArchRule use_cases_live_in_application = classes()
            .that().haveSimpleNameEndingWith("UseCase")
            .should().resideInAPackage("..application..");

    @ArchTest
    static final ArchRule controllers_live_in_api = classes()
            .that().haveSimpleNameEndingWith("Controller")
            .should().resideInAPackage("..api.http..");
}
```

Lưu ý: rule chỉ chặn được những gì nó mô tả. Mỗi khi review phát hiện một loại vi phạm mới lặp lại, **SHOULD** thêm rule tương ứng vào đây thay vì chỉ nhắc trong comment review.

### E2. Spring Modulith: boundary giữa các feature

ArchUnit khó diễn đạt rule "không import package con của feature khác" một cách tổng quát. Spring Modulith làm việc này theo quy ước: root package của feature là API, package con là internal.

```java
package com.company.app.architecture;

import com.company.app.Application;
import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;
import org.springframework.modulith.docs.Documenter;

class ModularityTest {

    static final ApplicationModules modules = ApplicationModules.of(Application.class);

    @Test
    void module_boundaries_are_respected() {
        modules.verify();   // fail nếu có cycle hoặc truy cập type internal của module khác
    }

    @Test
    void writes_module_documentation() {
        new Documenter(modules).writeDocumentation();   // sinh module map vào build/spring-modulith-docs
    }
}
```

Bổ sung:

- Giới hạn dependency được phép bằng `@ApplicationModule(allowedDependencies = {"customer", "common"})` trong `package-info.java` của feature. Điều này biến mục "Dependencies" trong README thành rule kiểm tra được.
- `common` khai báo là module mở (`type = ApplicationModule.Type.OPEN`) để các feature dùng được package con của nó.
- Test tích hợp một module riêng lẻ bằng `@ApplicationModuleTest`.

### E3. `verify.sh`: một lệnh duy nhất

Cấu hình Gradle tách unit test và integration test bằng JUnit tag:

```kotlin
// build.gradle.kts
tasks.test {
    useJUnitPlatform { excludeTags("integration") }
}

val integrationTest by tasks.registering(Test::class) {
    useJUnitPlatform { includeTags("integration") }
    testClassesDirs = sourceSets.test.get().output.classesDirs
    classpath = sourceSets.test.get().runtimeClasspath
    shouldRunAfter(tasks.test)
}

tasks.check { dependsOn(integrationTest) }
```

```bash
#!/usr/bin/env bash
# scripts/verify.sh
# Usage:
#   ./scripts/verify.sh          Full verification. MUST pass before a PR.
#   ./scripts/verify.sh --fast   Format + unit + architecture tests. For the inner loop.
set -euo pipefail
cd "$(dirname "$0")/.."

if [[ "${1:-}" == "--fast" ]]; then
  ./gradlew spotlessApply test
  exit 0
fi

./gradlew check                       # spotlessCheck, checkstyle, unit, architecture, integration
./scripts/check-business-rules.sh
echo "VERIFY: PASSED"
```

`check` đã bao gồm `test`, nên không gọi `clean test` riêng (tránh chạy test hai lần). Không dùng `clean` mặc định vì làm mất build cache; chỉ dùng khi nghi ngờ cache hỏng.

### E4. Kiểm tra traceability của Business Rule

```bash
#!/usr/bin/env bash
# scripts/check-business-rules.sh
# Fails if:
#  - a rule documented in a feature README has no test tagged with it
#  - a rule ID used in code or tests is not documented in any README
set -euo pipefail
cd "$(dirname "$0")/.."

PATTERN='BR-[A-Z]+-[0-9]{3}'

extract() { grep -rhoE "$PATTERN" "$@" 2>/dev/null | sort -u || true; }

documented=$(grep -rhE "^\| *$PATTERN" --include=README.md src/main/java 2>/dev/null \
             | grep -v '(retired)' | grep -oE "$PATTERN" | sort -u || true)
all_documented=$(extract --include=README.md src/main/java)
tested=$(extract --include='*.java' src/test/java)
in_code=$(extract --include='*.java' src/main/java)

missing_tests=$(comm -23 <(echo "$documented") <(echo "$tested") | sed '/^$/d')
undocumented=$(comm -13 <(echo "$all_documented") <(printf '%s\n%s\n' "$tested" "$in_code" | sort -u) | sed '/^$/d')

status=0
if [[ -n "$missing_tests" ]]; then
  echo "Business rules documented but not tested:"; echo "$missing_tests" | sed 's/^/  /'; status=1
fi
if [[ -n "$undocumented" ]]; then
  echo "Business rule IDs used but not documented in any README:"; echo "$undocumented" | sed 's/^/  /'; status=1
fi
[[ $status -eq 0 ]] && echo "Business rules: OK"
exit $status
```

### E5. CI pipeline

```text
PR opened
  ├── ./scripts/verify.sh                       (bắt buộc)
  ├── OpenAPI diff vs main                      (fail nếu breaking change chưa được duyệt)
  ├── Flyway: không cho sửa file migration đã tồn tại trên main
  ├── Dependency vulnerability scan
  └── CODEOWNERS review
```

Kiểm tra migration bất biến có thể làm đơn giản:

```bash
# Fail nếu PR sửa hoặc xóa migration đã có trên main
changed=$(git diff --name-only --diff-filter=MD origin/main...HEAD -- src/main/resources/db/migration)
if [[ -n "$changed" ]]; then echo "Existing migrations must not change:"; echo "$changed"; exit 1; fi
```

### E6. CODEOWNERS và PR template

```text
# CODEOWNERS
/src/main/java/com/company/app/order/      @team-checkout
/src/main/java/com/company/app/payment/    @team-payments
/src/main/java/com/company/app/common/     @platform-architecture
/src/test/java/com/company/app/architecture/  @platform-architecture
/src/test/resources/archunit_store/        @platform-architecture
/docs/adr/                                 @platform-architecture
/AGENTS.md                                 @platform-architecture
```

Việc đặt `architecture/` và `archunit_store/` dưới quyền owner riêng đảm bảo không ai (người hay Agent) có thể nới lỏng rule kiến trúc mà không qua review.

```md
<!-- .github/pull_request_template.md -->
## What & why

## Business rules affected
<!-- BR IDs, or "none" -->

## Checklist
- [ ] ./scripts/verify.sh passes
- [ ] Feature README updated (rules, use cases, events) or not needed
- [ ] New migration added for schema changes (no existing migration edited)
- [ ] ADR added/updated for architectural changes
- [ ] No breaking API change, or new version created
- [ ] Authored or co-authored by an AI agent: yes / no
```

---

## Phần F. Testing

### F1. Cấu trúc

Test theo cùng cấu trúc feature và layer với source, để Agent tìm test bằng cùng mental model:

```text
src/test/java/com/company/app/
├── architecture/
│   ├── ArchitectureTest.java
│   └── ModularityTest.java
└── order/
    ├── domain/            OrderTest.java
    ├── application/       CreateOrderUseCaseTest.java
    ├── api/http/          OrderControllerTest.java
    ├── infrastructure/    JpaOrderRepositoryIT.java
    └── fixtures/          OrderFixtures.java, CustomerSnapshots.java
```

### F2. Chiến lược theo layer

| Layer | Loại test | Công cụ | Tốc độ | Tỉ trọng |
|---|---|---|---|---|
| domain | Unit thuần, không Spring | JUnit 5, AssertJ | ms | Nhiều nhất. Mọi BR-* đều test ở đây nếu rule nằm trong domain |
| application | Unit với fake/in-memory port | JUnit 5, fake `OrderRepository` | ms | Mỗi use case: happy path + nhánh lỗi + authorization |
| api | Slice test | `@WebMvcTest`, MockMvc | giây | Mapping request/response, status code, validation |
| infrastructure | Integration với DB thật | `@DataJpaTest` + Testcontainers, tag `integration` | giây | Mapping, query, `@Version` |
| module | Module integration | `@ApplicationModuleTest` | giây | Luồng event giữa module |
| end-to-end | Toàn ứng dụng | `@SpringBootTest` + Testcontainers | chậm | Ít, chỉ luồng then chốt |

Quy định:

- Integration test **MUST** dùng database thật qua Testcontainers, **MUST NOT** dùng H2 thay PostgreSQL (khác dialect gây pass giả).
- Application test **SHOULD** dùng fake implementation thay vì mock từng method, để test mô tả hành vi thay vì mô tả cách gọi.
- Test data **SHOULD** tạo qua fixture/builder trong `fixtures/`, không lặp lại constructor dài trong từng test.

### F3. Đặt tên test

Tên test mô tả hành vi, không mô tả method:

| Tránh | Dùng |
|---|---|
| `testCreateOrder1` | `active_customer_can_create_order` |
| `testCancel` | `completed_order_cannot_be_cancelled` |

Test cho business rule có thêm `@Tag("BR-...")` (D3). Test integration có thêm `@Tag("integration")`.

### F4. Agent và test

- Agent **MUST** thêm hoặc cập nhật test cho mọi thay đổi hành vi.
- Agent **MUST NOT** sửa assertion của test đang fail để build pass, trừ khi task yêu cầu đổi chính hành vi đó và README đã được cập nhật tương ứng.
- Với bug fix, Agent **SHOULD** viết test tái hiện bug trước, xác nhận nó fail, rồi mới sửa.

---

## Phần G. Áp dụng cho codebase hiện hữu (brownfield)

Phần lớn hệ thống lớn không bắt đầu từ đầu. Áp dụng standard theo kiểu "big bang refactor" gần như chắc chắn thất bại. Lộ trình dưới đây cho phép áp dụng dần mà vẫn release bình thường.

### G0. Baseline: chặn mọi thứ tệ hơn (1–2 tuần)

1. Thêm `scripts/verify.sh` bọc lệnh build hiện có.
2. Viết `AGENTS.md` mô tả **đúng hiện trạng**, kể cả phần chưa tốt, ví dụ: "Legacy code ở `com.company.app.service` theo technical layer. Code mới đặt theo feature trong `com.company.app.<feature>`."
3. Thêm ArchUnit với `FreezingArchRule`: vi phạm hiện có được ghi vào `archunit_store` và cho qua; vi phạm **mới** làm build fail.

```java
@ArchTest
static final ArchRule no_generic_names = FreezingArchRule.freeze(
        noClasses().should().haveNameMatching(".*(Utils|Helper|Manager|ServiceImpl)$"));
```

```properties
# src/test/resources/archunit.properties
freeze.store.default.path=src/test/resources/archunit_store
freeze.store.default.allowStoreCreation=true
freeze.refreeze=false
```

Khi một vi phạm cũ được sửa, ArchUnit tự xóa nó khỏi store. Số dòng trong store là chỉ số nợ kiến trúc, chỉ được phép giảm.

### G1. Chọn feature ưu tiên theo mức độ thay đổi

Không refactor theo thứ tự alphabet. Refactor nơi code thay đổi nhiều nhất, vì đó là nơi Agent làm việc nhiều nhất:

```bash
git log --since="6 months ago" --name-only --format= -- src/main/java \
  | sort | uniq -c | sort -rn | head -30
```

### G2. Di chuyển cơ học (theo từng feature)

1. Tạo package `<feature>/` và di chuyển class liên quan vào bằng refactor của IDE. **Không đổi logic** trong cùng PR.
2. Viết feature README, gán BR ID cho các rule đã biết, ghi "Known quirks".
3. Thêm test cho các BR ID còn thiếu (characterization test: ghi lại hành vi hiện tại, kể cả khi nó kỳ lạ).

### G3. Tách lớp dần (boy scout rule)

Mỗi khi task chạm vào một feature đã di chuyển:

- Tách use case khỏi `XxxService` lớn: một use case mới cho mỗi method được sửa.
- Đẩy rule nghiệp vụ từ service vào aggregate trong `domain/`.
- Thay import trực tiếp sang feature khác bằng public API hoặc event.

### G4. Siết dần enforcement

Khi một feature đã sạch, bật rule không freeze cho riêng package đó, rồi cuối cùng bật `ApplicationModules.verify()` cho toàn bộ ứng dụng.

| Giai đoạn | Tiêu chí hoàn thành |
|---|---|
| G0 | `verify.sh` chạy trong CI; ArchUnit freeze hoạt động; `AGENTS.md` có |
| G1–G2 | Top 5 feature thay đổi nhiều nhất có package riêng và README |
| G3 | Top 5 feature có use case class và domain chứa rule |
| G4 | `archunit_store` rỗng; Modulith verify pass |

---

## Phần H. Governance & metrics

### H1. Ai duy trì gì

| Tài sản | Owner | Khi nào cập nhật |
|---|---|---|
| Tài liệu standard này | `@platform-architecture` | Review định kỳ mỗi quý |
| `AGENTS.md` root | `@platform-architecture` | Khi đổi quy ước, công cụ, lệnh build |
| Feature README | Team sở hữu feature | **Trong cùng PR** với thay đổi rule, use case, event |
| ADR | Người đề xuất quyết định | Khi có quyết định kiến trúc mới hoặc thay thế |
| ArchUnit / Modulith tests | `@platform-architecture` | Khi phát hiện loại vi phạm mới lặp lại |

Tài liệu được cập nhật trong cùng PR với code, không phải "để sau". Reviewer **SHOULD** từ chối PR thay đổi hành vi mà không cập nhật README tương ứng.

### H2. Đo lường hiệu quả

Không đo thì không biết standard có giúp Agent tốt hơn hay chỉ thêm thủ tục.

| Metric | Cách đo | Hướng mong muốn |
|---|---|---|
| First-pass verify rate | % PR do Agent tạo pass `verify.sh` ở lần push đầu | Tăng |
| Architecture review comments | Số comment review về vị trí code, layer, naming | Giảm |
| Rework rate | % PR của Agent cần sửa lớn sau review | Giảm |
| Files read per task | Số file Agent mở trước khi sửa (từ log của công cụ, nếu có) | Giảm |
| Frozen violations | Số dòng trong `archunit_store` | Giảm đều, không bao giờ tăng |
| BR coverage | % BR có test (output của `check-business-rules.sh`) | 100% |
| Time to first PR (người mới) | Ngày từ onboarding đến PR đầu tiên được merge | Giảm |

Lấy baseline trước G0, đo lại hàng tháng. Nếu sau 2–3 tháng các chỉ số không cải thiện, xem lại standard thay vì tiếp tục áp đặt.

---

## Phụ lục

### P1. Golden path cho Agent

```text
TASK
 │
 ▼
AGENTS.md ─────────────── (rule cứng, điều cấm, khi nào dừng)
 │
 ▼
Xác định FEATURE
 │
 ▼
<feature>/README.md ───── (BR ID, use case, event, quirks)
 │
 ▼
grep BR ID → test hiện có
 │
 ▼
APPLICATION use case → DOMAIN aggregate
 │
 ▼
Viết/cập nhật TEST (fail trước nếu là bug fix)
 │
 ▼
IMPLEMENT thay đổi nhỏ nhất
 │
 ▼
Cập nhật README / ADR nếu cần
 │
 ▼
./scripts/verify.sh ── fail? → sửa; fail không liên quan? → DỪNG, báo người
 │
 ▼
REPORT theo format trong AGENTS.md
```

Nếu Agent phải grep toàn repository, đoán service, mở 30 class, đoán transaction, thì repository chưa đạt standard.

### P2. Checklist

**MUST** (thiếu là chưa đạt):

- [ ] Package by feature; chiến lược module được ghi trong ADR-001.
- [ ] `AGENTS.md` ở root, dưới ~150 dòng, có mục cấm và mục dừng lại hỏi.
- [ ] Mỗi feature có `README.md` theo template D2.
- [ ] Inbound adapter ở `api/`, outbound adapter ở `infrastructure/`.
- [ ] Business rule nằm trong `domain/`; domain không phụ thuộc framework (theo ADR-003).
- [ ] `@Transactional` và authorization chỉ ở `application/`.
- [ ] Giao tiếp giữa feature chỉ qua public API ở root package, domain event, hoặc `common/`.
- [ ] Business rule quan trọng có BR ID ở README, code và test; CI kiểm tra traceability.
- [ ] ArchUnit (và Modulith nếu dùng) chạy trong `verify.sh`.
- [ ] Một lệnh `./scripts/verify.sh` duy nhất, chạy trong CI.
- [ ] Schema thay đổi qua migration; migration cũ bất biến, có kiểm tra CI.
- [ ] Lỗi HTTP trả `ProblemDetail` với code ổn định, map ở một nơi.
- [ ] `CODEOWNERS` bảo vệ `common/`, `architecture/`, `archunit_store/`, `docs/adr/`, `AGENTS.md`.

**SHOULD**:

- [ ] Spring Modulith cho hệ thống vừa và lớn.
- [ ] Read side dùng projection, không load aggregate.
- [ ] Integration event qua outbox; consumer idempotent.
- [ ] OpenAPI diff trong CI.
- [ ] Integration test dùng Testcontainers.
- [ ] Metrics ở H2 được đo định kỳ.

### P3. Lịch sử thay đổi

**2.2**: Tách nội dung microservices sang tài liệu riêng `02-spring-boot-ai-agent-microservices.md`. Tài liệu này chỉ còn phạm vi một deployable.

**2.1**: Thêm nội dung microservices (nay đã chuyển sang tài liệu 02).

**2.0** so với 1.0:

| Vấn đề ở v1 | Thay đổi ở v2 |
|---|---|
| Kafka listener / job không có chỗ hợp lệ trong dependency rule | Tách `api/` = inbound, `infrastructure/` = outbound (B3) |
| `api → domain` không rõ | Bảng dependency đầy đủ (B5) |
| Rule cross-feature mâu thuẫn với ví dụ BR-ORDER-001 | Public API ở root package + port `CustomerDirectory` (B5) |
| Package-by-feature không enforce boundary | Chọn cơ chế theo quy mô, mặc định Spring Modulith (B1, E2) |
| Tách domain/JPA không bàn chi phí, mất `@Version` | Hai biến thể + ADR; `create` / `reconstitute` mang version (B6) |
| ArchUnit chỉ kiểm tra 2 package | Bộ rule đầy đủ cho layer, transaction, naming (E1) |
| Source of truth có hai thứ tự trái ngược | Một thứ tự duy nhất + quy trình dừng khi mâu thuẫn (D5) |
| Không có guardrail cho Agent | Mục MUST NOT, stop conditions, report format trong `AGENTS.md` (D1) |
| Tài liệu dễ lỗi thời, không có cơ chế kiểm tra | `check-business-rules.sh`, Modulith Documenter, CODEOWNERS (E4, D6, E6) |
| Error, security, event, observability chỉ được nhắc tên | Convention cụ thể (Phần C) |
| `verify.sh` chạy test hai lần, `set -e` thiếu | `set -euo pipefail`, chế độ `--fast` (E3) |
| Không có lộ trình cho codebase hiện hữu | FreezingArchRule + lộ trình G0–G4 (Phần G) |
| Không đo hiệu quả | Metrics và owner (Phần H) |
| Heading không nhất quán, nhiều block code một từ | Tái cấu trúc theo Phần A–H, dùng bảng thay block rời |
