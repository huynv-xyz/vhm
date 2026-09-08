# NOXH Java Platform

**Phạm vi:** các dịch vụ Java/Spring Boot NOXH · Java 25 / Spring Boot 4
**Đối chiếu:** `vhm-dossier-core`, `vhm-ocr-ekyc` (cũ) ↔ `new-structure` (mới) · Số liệu đo 07/09/2026

---

## 1. Tóm tắt

**Vấn đề.** Mỗi service tự mang một bản sao nền tảng Spring Boot: dependency Maven, cấu hình JPA/Kafka/Redis, REST client, exception handler, response envelope, base entity, UUID generator, logging, security, utility. Không bản nào được công nhận là chính thức, nên mỗi bản vá hoặc thay đổi contract upstream trở thành N thay đổi trên N repository, kèm rủi ro bỏ sót.

**Giải pháp.** Tách theo ranh giới sở hữu: `libraries` (4 artifact nền tảng, có version và owner) + `service` (nghiệp vụ). Nguyên tắc bất biến: **nghiệp vụ của service nào thuộc service đó**.

| Chỉ số | Cũ | Mới | Δ |
|---|---:|---:|---:|
| Dossier — Java files (`src/main`) | 310 | 221 | −89 |
| Dossier — Java LOC | 26.463 | 21.611 | −18% |
| Dossier — POM | 436 dòng | 51 dòng | −88% |
| OCR/eKYC — Java files | 106 | 66 | −40 |
| OCR/eKYC — Java LOC | 5.695 | 4.206 | −26% |
| OCR/eKYC — POM | 124 dòng | 19 dòng | −85% |
| Thrift generated code trong service | ~66.500 dòng/service dùng Profile | 0 | về `vhm-client` |

---

## 2. Nhược điểm cấu trúc cũ

### 2.1. Service gánh hai nhiệm vụ

Package `core` của dossier chứa đồng thời nghiệp vụ và hạ tầng, không phân tầng:

```text
vn/vinhomes/agent/dossier/core/
├── annotation/  config/  security/  utils/  validation/   ← hạ tầng
├── exception/   kafka/   metrics/   mapper/               ← hạ tầng
├── client/                                                ← outbound integration
├── controller/  dto/  model/  repository/  service/       ← nghiệp vụ
└── event/  notification/  enums/  constant/               ← nghiệp vụ
```

Dưới `core` cùng tồn tại REST client cho File/OCR/VinBigData/Market/Message Delivery; Thrift runtime + connection pool + generated contract; Kafka config và legacy Kafka config; Redis/Redisson config; HMAC filter; logging interceptor, masking layout; base entity, UUID generator, API response, exception handler. OCR/eKYC lặp lại đúng mô hình đó ở quy mô nhỏ hơn.

→ Nhìn vào package không phân biệt được đâu là nghiệp vụ riêng, đâu là nền tảng dùng lại được.

### 2.2. Trùng tên nhưng không đảm bảo trùng hành vi

Ít nhất **12 tên class xuất hiện ở cả hai repository**:

```text
AppErrorCode   BaseEntity   BaseEntityUUID   HealthController   OutboxEvent
UUIDv7Generated   UUIDv7Generator   RestControllerExceptionHandler
PrepareDownloadRequest   PrepareUploadRequest   OcrResponse
```

Hai class cùng tên có thể khác field, validation, status code, serialization. Đây là dạng duplication nguy hiểm hơn copy-paste thuần: developer import nhầm và chỉ phát hiện khi tích hợp hoặc chạy production.

### 2.3. Client khó tìm, khó tái sử dụng

Client cũ nằm trong namespace service sở hữu (`vn.vinhomes.agent.dossier.core.client`). Service khác không biết đã có client sẵn chưa nên tạo mới hoặc copy; bug fix và thay đổi contract phải làm nhiều nơi; DTO upstream nằm trong domain package dossier khiến người đọc hiểu nhầm là model nghiệp vụ.

Thrift nặng hơn: riêng generated code cho Profile chiếm **~66.500 dòng** — mỗi service cần Profile đều phải mang IDL và sinh lại toàn bộ khối này.

### 2.4. Maven và version phân tán

POM dossier 436 dòng, OCR 124 dòng. Mỗi repo tự quản Spring Boot starters, annotation processor, SpringDoc, Lombok, MapStruct, version override để vá CVE, plugin compiler/Surefire/packaging.

→ **Hiện không có cách trả lời nhanh "còn service nào chưa được vá?"** ngoài mở từng repository kiểm tra thủ công.

### 2.5. Cấu hình lặp và khó xác định nguồn

Cùng một Kafka bootstrap server được biểu diễn qua cả `spring.kafka.*` lẫn `kafka.*`. Client config dùng 6 root khác nhau: `file-client`, `ocr-client`, `market-client`, `vin-bigdata-client`, `external.message-delivery`, `thrift.client`.

→ Khó biết property nào thực sự được bind; đổi tên property có thể khiến service khởi động với default sai mà không báo lỗi.

### 2.6. Onboarding phụ thuộc người hướng dẫn

Người mới phải hỏi trước khi viết được dòng nghiệp vụ đầu tiên: API response chuẩn ở đâu, exception nào map status code nào, entity kế thừa class nào, client gọi File Service đã có chưa, Kafka listener cần factory tên gì. Câu trả lời phụ thuộc repository đang mở → năng lực bàn giao gắn với cá nhân thay vì gắn với tài sản kỹ thuật.

### 2.7. Chi phí nếu giữ nguyên

Chi phí không tăng theo số dòng code mà theo **số biến thể**: vá một CVE phải sửa và test riêng từng repo; upstream đổi contract phải rà soát thủ công toàn bộ repo với rủi ro bỏ sót; service mới lại copy nền tảng từ repo gần nhất, sinh thêm một biến thể phải duy trì.

---

## 3. Nguyên nhân gốc

> **Chưa có ranh giới sở hữu giữa phần dùng chung và phần nghiệp vụ.**

Không có ranh giới thì mọi thành phần kỹ thuật mặc định thuộc về repository nào tình cờ cần nó trước. Không bản nào là chính thức, nên sao chép là phương án hợp lý nhất với từng developer — dù đắt nhất với tổ chức.

Năm nguyên tắc của cấu trúc mới:

- **P1** — Service chỉ sở hữu nghiệp vụ của chính nó, không tự chứa lại base entity, API envelope, global exception handler, generic paging, Kafka/Redis boilerplate.
- **P2** — Library không chứa business rule. Một class chỉ vào library khi có ≥2 consumer thực tế hoặc là capability nền tảng, không phụ thuộc model nghiệp vụ, có owner, và việc dùng chung **giảm tổng độ phức tạp** chứ không chỉ chuyển code sang chỗ khác.
- **P3** — Default ở library, giá trị môi trường ở deployment: `Environment/Secret` → `application.yml` của service → defaults có tên riêng của library (`vhm-common-defaults.yml`, `vhm-web-defaults.yml`, `vhm-client-defaults.yml`).
- **P4** — Auto-configuration phải có điều kiện. Service không dùng Redis/Thrift/HMAC không được fail khởi động chỉ vì default property rỗng. Đây là nguyên tắc then chốt để library dùng chung không thành *monolith dùng chung*.
- **P5** — Cấu hình local không rò vào test; job nền phải có công tắc kích hoạt tường minh.

---

## 4. Cấu trúc mới

```text
new-structure/
├── libraries/
│   ├── vhm-spring-boot-parent/   # build convention, không chứa Java code
│   ├── vhm-common/               # primitive/hạ tầng, không phụ thuộc web
│   ├── vhm-web-starter/          # chuẩn inbound HTTP
│   └── vhm-client/               # outbound integration (+ src/main/thrift)
└── service/
    ├── vhm-dossier-core/
    ├── vhm-ocr-ekyc/
    └── vhm-campaign-core/
```

| Module | Chịu trách nhiệm | **Không** chịu trách nhiệm |
|---|---|---|
| `vhm-spring-boot-parent` | Java/Spring Boot baseline; version dependency và bản vá CVE; compiler, annotation processor, Surefire, packaging plugin | Business dependency của một service; kéo mọi thư viện runtime vào mọi service để POM ngắn |
| `vhm-common` | `BaseEntity`, UUIDv7, repository primitive; utility thuần đã có test; signing/cipher primitive; default datasource, JPA, virtual threads, structured logging; `logback-spring.xml` canonical | Controller, HTTP response, servlet filter; client của upstream cụ thể; domain status, business error code |
| `vhm-web-starter` | API response envelope, paging; global exception handler; Spring MVC, validation, OpenAPI defaults; security filter chain, actor context | Business exception code; REST client gọi ra ngoài; JPA entity/repository; logic notification/assignment/workflow |
| `vhm-client` | `RestClient` infrastructure; typed client File/OCR/VinBigData/Market/Message Delivery/Profile; DTO theo contract upstream; authentication, timeout, retry; Thrift IDL, generated code, transport pool | Quyết định *khi nào* gọi client; map kết quả thành trạng thái domain; fallback nghiệp vụ; phụ thuộc `vhm-web-starter` |

Ví dụ ranh giới: `FileClient.prepareUpload()` thuộc library — quyết định hồ sơ nào upload loại tài liệu nào thuộc dossier. `ProfileClient` thuộc library — quyết định reviewer nào được phân công thuộc dossier.

**Service** giữ toàn bộ domain entity, business rule, workflow, migration và adapter đặc thù. FPT OCR client tiếp tục nằm trong OCR service nếu chỉ OCR dùng — **không đưa mọi outbound call vào `vhm-client` chỉ vì tên class có chữ `Client`**.

### 4.1. Dependency rules

```text
vhm-common ← vhm-web-starter
           ← vhm-client

service → common + web-starter (nếu có HTTP API) + client capability cần dùng
```

- `vhm-common` **không** phụ thuộc web starter hoặc client.
- `vhm-client` **không** phụ thuộc `vhm-web-starter` — đã gỡ trong PoC, nhờ vậy worker/consumer dùng outbound client không bị kéo theo MVC, controller, web security.
- Library **không** phụ thuộc service. Không dependency cycle. Không tạo bean/connection cho capability đang tắt.

### 4.2. Bảng quyết định đặt code

Áp dụng theo thứ tự, dừng ở dòng đầu tiên khớp:

| # | Điều kiện | Vị trí |
|---|---|---|
| 1 | Chứa thuật ngữ, trạng thái hoặc quyết định của một domain | Service sở hữu domain |
| 2 | Mô tả contract upstream, không quyết định business flow | `vhm-client` |
| 3 | Xử lý inbound HTTP/API nhất quán giữa các service | `vhm-web-starter` |
| 4 | Primitive/utility, không phụ thuộc HTTP và không mang nghiệp vụ | `vhm-common` |
| 5 | Chỉ quản version/plugin/build | `vhm-spring-boot-parent` |
| 6 | Chưa có consumer thứ hai và chưa phải nền tảng bắt buộc | Giữ tại service |

---

## 5. Ưu điểm

| Nhược điểm cũ | Cách xử lý | Kết quả đo được |
|---|---|---|
| Nền tảng bị xây lặp từng repo | Hạ tầng về library có owner và test | −89 file (dossier), −40 file (OCR) |
| Không rõ client chính thức | Gom outbound contract vào `vhm-client` | ~66.500 dòng generated code không còn nhân bản |
| Build/dependency phân tán | Parent quản version/plugin | POM −85…88%; vá CVE tập trung, đo adoption qua version |
| Cấu hình khó dự đoán | Default library → override service → local profile riêng | Test tái lập, ít phụ thuộc máy dev |
| Nghiệp vụ lẫn hạ tầng | Service chỉ giữ domain | Reviewer xác định phạm vi ảnh hưởng qua module |
| Onboarding truyền miệng | Cấu trúc + owner + guide cạnh code | Người mới tự tìm code bằng cấu trúc |

Luồng làm việc trước / sau:

| Nhu cầu | Cũ | Mới |
|---|---|---|
| Tạo API mới | Tìm/copy response, exception, validation pattern trong repo | Dùng contract + handler từ `vhm-web-starter` |
| Tạo entity | Tìm base entity/UUID generator đúng phiên bản | Dùng persistence primitive từ `vhm-common` |
| Gọi File/Profile/Market | Search nhiều package/repo, nguy cơ tạo client mới | Tra `vhm-client`, inject typed client |
| Vá lỗi client | Sửa từng bản copy, đồng bộ thủ công | Sửa library, test một contract, nâng version |
| Nâng dependency/CVE | Audit và sửa nhiều POM | Version tập trung; dependency riêng vẫn ở đúng service |

Khác biệt cốt lõi: developer chuyển từ **"tìm một repo để copy"** sang **"dùng một contract có owner và version"**. Với quản lý: đo adoption qua library version thay vì audit thủ công, rollback bằng cách pin lại version trước.

---

## 6. Kết quả kiểm chứng

Cấu trúc mới đã được dựng và chạy thật, không dừng ở phân tích trên giấy:

| Hạng mục | Kết quả |
|---|---|
| Libraries reactor | `mvn clean install` — thành công, gồm test từng module |
| OCR/eKYC | Full `mvn verify` — thành công |
| OCR persistence | PostgreSQL Testcontainers + Liquibase — thành công |
| Dossier | `mvn -DskipTests package` — thành công với dependency mới |
| Dependency direction | `vhm-client` không còn phụ thuộc `vhm-web-starter` |
| Capability isolation | OCR không khởi tạo Thrift/Profile pool khi capability tắt |
| Business leakage audit | Library production source không còn tham chiếu dossier/NOXH/OCR-eKYC/PTT |

**Ba lưu ý khi đọc số liệu §1:**

1. Code không "biến mất": phần dùng chung chuyển sang libraries, phần trùng lặp bị loại bỏ.
2. POM dossier cố ý giữ 51 dòng để khai báo minh bạch dependency riêng (Syncfusion, POI, Thymeleaf phục vụ xuất báo cáo). Đưa chúng vào parent chỉ để POM ngắn hơn sẽ làm **mọi** service mang classpath không cần thiết.
3. `vhm-client` có LOC lớn do code sinh từ Thrift — không dùng con số này để đánh giá độ phức tạp do developer tự viết.
