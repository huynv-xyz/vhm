## 1. Tóm tắt

**Vấn đề.** Mỗi service tự mang theo một bản sao nền tảng Spring Boot: dependency Maven, cấu hình JPA/Kafka/Redis, REST client, exception handler, response envelope, base entity, UUID generator, logging, security, utility. Hệ quả không phải "trùng vài class" mà là: không có bản nào được công nhận là chính thức, nên mỗi bản vá hoặc thay đổi contract upstream trở thành N thay đổi trên N repository, kèm rủi ro bỏ sót.

**Giải pháp.** Tách theo ranh giới sở hữu, thành hai nhóm và bốn artifact nền tảng:

| Artifact | Vai trò |
|---|---|
| `vhm-spring-boot-parent` | Build convention: Java/Spring Boot baseline, version, plugin |
| `vhm-common` | Primitive/utility/hạ tầng dùng chung, không phụ thuộc web |
| `vhm-web-starter` | Chuẩn inbound HTTP: response, error, validation, web security |
| `vhm-client` | Outbound integration: typed client + contract của upstream |

Service chỉ giữ nghiệp vụ và adapter đặc thù. Nguyên tắc bất biến: **nghiệp vụ của service nào thuộc service đó**.

**Kết quả PoC.**

| Chỉ số | Cũ | Mới | Thay đổi |
|---|---:|---:|---:|
| Dossier — Java files (`src/main`) | 310 | 221 | −89 file |
| Dossier — Java LOC | 26.463 | 21.611 | −18% |
| Dossier — POM | 436 dòng | 51 dòng | −88% |
| OCR/eKYC — Java files (`src/main`) | 106 | 66 | −40 file |
| OCR/eKYC — Java LOC | 5.695 | 4.206 | −26% |
| OCR/eKYC — POM | 124 dòng | 19 dòng | −85% |
| Thrift generated code phải duy trì trong service | ~66.500 dòng/service dùng Profile | 0 | Tập trung tại `vhm-client` |

Libraries reactor build/test thành công; OCR/eKYC qua full `mvn verify` gồm PostgreSQL Testcontainers; dossier compile/package thành công với dependency mới.

**Cần quyết định.** Chốt boundary + ownership, cho phép hardening và publish artifact nội bộ, chọn OCR/eKYC làm pilot. Không migrate đồng loạt; rollout bắt buộc chỉ bắt đầu sau khi đạt quality gate ở §12.

---

## 2. Phân tích hiện trạng

### 2.1. Phương pháp đánh giá

Kết luận dựa trên đối chiếu trực tiếp hai repository cũ, phân loại source theo trách nhiệm, migrate thử sang cấu trúc mới và chạy build/test — không dựa trên tên package hay cảm nhận về độ gọn.

Tiêu chí xác định một thành phần có nên dùng chung:

- Có mang business rule của dossier/OCR không?
- Có consumer thực tế hoặc là capability nền tảng rõ ràng không?
- Có thể version, test và vận hành độc lập không?
- Khi service không dùng capability, dependency/bean/connection có bị loại bỏ không?
- Tổng độ phức tạp toàn hệ thống có giảm, hay chỉ chuyển vị trí code?

### 2.2. Service đang gánh hai nhiệm vụ

Package `core` của `vhm-dossier-core` cũ chứa đồng thời nghiệp vụ và hạ tầng, không phân tầng:

```text
vn/vinhomes/agent/dossier/core/
├── annotation/   config/   security/   utils/   validation/   ← hạ tầng
├── client/                                                     ← outbound integration
├── exception/    kafka/    metrics/    mapper/                 ← hạ tầng
├── controller/   dto/      model/      repository/  service/   ← nghiệp vụ
├── event/        notification/         enums/       constant/  ← nghiệp vụ
```

Cụ thể, dưới `core` cùng tồn tại: controller/service/repository/entity nghiệp vụ hồ sơ; REST client cho File, OCR, VinBigData, Market, Message Delivery; Thrift runtime + connection pool + factory + generated contract cho Profile Middleware; Kafka configuration và legacy Kafka configuration; Redis/Redisson configuration; security configuration, HMAC filter, actor context; logging interceptor, request tracking, masking layout, JSON/date helper; base entity, UUID generator, page DTO/mapper, API response, exception handler.

`vhm-ocr-ekyc` lặp lại đúng mô hình đó ở quy mô nhỏ hơn: tự có base controller, health controller, API response, exception handler, base entity, UUIDv7 generator, repository base, `FileUtil`/`HashUtil`/`HttpUtil`/`JsonUtil`/`StringUtil`/`UuidUtil`, cấu hình JPA/OpenAPI/RestClient/Scheduling/payload encryption, và File Management client riêng.

**Hệ quả:** nhìn vào package không phân biệt được đâu là năng lực nghiệp vụ riêng, đâu là nền tảng dùng lại được. Review phải tự phân loại thủ công.

### 2.3. Trùng tên nhưng không đảm bảo trùng hành vi

Đối chiếu hai repository cũ cho thấy ít nhất **12 tên class xuất hiện ở cả hai phía**:

```text
AppErrorCode          BaseEntity              BaseEntityUUID
HealthController      RestControllerExceptionHandler
UUIDv7Generated       UUIDv7Generator         OutboxEvent
PrepareDownloadRequest  PrepareUploadRequest  OcrResponse
```

Trùng tên không đảm bảo trùng contract — hai class cùng tên có thể khác field, validation, status code, serialization hoặc cách xử lý null. Đây là dạng duplication nguy hiểm hơn copy-paste thuần túy: developer dễ import nhầm và chỉ phát hiện khi tích hợp hoặc chạy production.

### 2.4. Client khó tìm, khó tái sử dụng

Client cũ nằm trong namespace của service sở hữu, ví dụ `vn.vinhomes.agent.dossier.core.client`. Ba vấn đề phát sinh:

1. Service khác không biết đã có client sẵn hay chưa.
2. Copy client sang repository mới → bug fix và thay đổi contract phải làm nhiều nơi.
3. DTO của upstream nằm trong domain package của dossier, khiến người đọc hiểu nhầm đó là model nghiệp vụ của dossier.

Thrift nặng hơn: file `.thrift`, generated source, transport pool, validation và wrapper client nằm xen kẽ code nghiệp vụ. Riêng phần generated cho Profile chiếm **~66.500 dòng** — ở cấu trúc cũ, mỗi service cần Profile đều phải mang IDL và sinh lại toàn bộ khối này trong repository của mình.

Ngoài ra rà soát phát hiện hai nhóm DTO/client File Service có contract gần giống nhau nhưng khác package và khác cách xử lý.

### 2.5. Maven và version dependency phân tán

POM dossier cũ 436 dòng, OCR 124 dòng. Mỗi repository tự khai báo/quản lý: Spring Boot starters, test libraries, annotation processors, SpringDoc/OpenAPI, Lombok, MapStruct, Hibernate processor, version override để xử lý CVE, plugin compiler/Surefire/packaging, repository nội bộ hoặc vendor.

Khi version được sửa ở một service nhưng chưa sửa ở service khác, hệ thống rơi vào trạng thái không đồng nhất. **Hiện không có cách trả lời nhanh câu hỏi "còn service nào chưa được vá?"** ngoài mở từng repository kiểm tra thủ công.

### 2.6. Cấu hình lặp và khó xác định nguồn

Mỗi service có thể đồng thời chứa `application.yml`, `application-local.yml`, biến môi trường và config class riêng. Cùng một Kafka bootstrap server được biểu diễn qua cả `spring.kafka.*` lẫn `kafka.*`. Client config từng dùng nhiều root khác nhau:

```text
file-client    ocr-client    market-client
vin-bigdata-client    external.message-delivery    thrift.client
```

Hệ quả: khó biết property nào thực sự được bind; default khác nhau giữa các service; đổi tên property có thể khiến service khởi động với default sai mà không báo lỗi; file local phình to vì chứa cấu hình kỹ thuật lặp lại.

### 2.7. Chi phí onboarding cao

Trước khi viết được dòng nghiệp vụ đầu tiên, người mới phải trả lời: API response chuẩn nằm ở đâu; exception nào map thành status code nào; entity mới kế thừa class nào và sinh UUID ra sao; dùng Spring `RestClient` hay client tự viết; client gọi File Service đã tồn tại chưa; Kafka listener cần factory tên gì; security/health/OpenAPI/logging copy từ repository nào; cấu hình local nằm trong YAML hay environment variable.

Nếu câu trả lời phụ thuộc vào repository đang mở thì tổ chức chưa có *paved road*, và năng lực bàn giao gắn với cá nhân thay vì gắn với tài sản kỹ thuật.

### 2.8. Lỗi chỉ lộ ra khi chạy

Các lỗi sau đều tồn tại sẵn trong cấu trúc cũ, chỉ phát hiện khi migrate và chạy test thật:

| Hiện tượng | Nguyên nhân gốc |
|---|---|
| Test OCR dùng nhầm database/Kafka của môi trường dev | File cấu hình local tự động rò vào test profile |
| Integration test không ổn định | Scheduled outbox publisher chạy trong test context và claim row trước test case |
| OCR khởi tạo Thrift transport pool dù không dùng Profile | Auto-configuration luôn chạy vì default đã trỏ localhost |
| Kafka listener không khởi động | Consumer yêu cầu bean `legacyManualCommitKafkaListenerContainerFactory`, shared config chưa có alias |
| Ứng dụng chết ngay giai đoạn init logging | `logback-spring.xml` còn tham chiếu `...core.config.tracing.MaskingPatternLayout` đã bị di chuyển |
| PostgreSQL báo role `myuser` không tồn tại | Default username/password trong service không khớp Docker Compose |

Điểm chung: **trách nhiệm và lifecycle của hạ tầng không được quản lý tại một nơi**. Mỗi service tự bảo đảm nhiều chi tiết kỹ thuật có quan hệ với nhau; khi một phần được di chuyển hoặc đổi tên, phần tham chiếu còn lại bị bỏ sót.

Các lỗi này là đầu vào trực tiếp cho nguyên tắc thiết kế §3.

### 2.9. Chi phí nếu giữ nguyên

Chi phí không tăng theo số dòng code mà theo **số biến thể**:

| Hạng mục | Chi phí hiện tại | Xu hướng |
|---|---|---|
| Vá một CVE chung | Sửa + test riêng từng repository | Tuyến tính theo số service |
| Upstream đổi contract | Rà soát thủ công toàn bộ repo, rủi ro bỏ sót | Tuyến tính, kèm rủi ro sự cố production |
| Khởi tạo service mới | Copy nền tảng từ repo gần nhất | Sinh thêm một biến thể phải duy trì |
| Onboarding | Phụ thuộc người hướng dẫn | Key-person risk khi đội mở rộng |
| Review | Reviewer tự phân biệt business vs hạ tầng | Chất lượng không ổn định |

---

## 3. Nguyên nhân gốc và nguyên tắc thiết kế

Toàn bộ vấn đề ở §2 quy về một nguyên nhân:

> **Chưa có ranh giới sở hữu (ownership boundary) giữa phần dùng chung và phần nghiệp vụ.**

Không có ranh giới thì mọi thành phần kỹ thuật mặc định thuộc về repository nào tình cờ cần nó trước. Không có bản nào được công nhận là chính thức, nên sao chép trở thành phương án hợp lý nhất với từng developer — dù là phương án đắt nhất với tổ chức.

Năm nguyên tắc của cấu trúc mới:

**P1 — Service chỉ sở hữu nghiệp vụ của chính nó.**
Được phép: domain entity/enum, use case service, business validation và error code, repository gắn schema của service, controller/DTO của API do service sở hữu, Kafka consumer/producer nghiệp vụ, migration/schema/template/workflow riêng, adapter cho upstream đặc thù chưa đủ điều kiện dùng chung.
Không nên tự chứa lại: base entity, UUID generator, API envelope, global exception handler, health controller, generic paging, generic REST setup, Kafka/Redis boilerplate, client của upstream đã chuẩn hóa.

**P2 — Library không được chứa business rule.**
Một class chỉ vào library khi: có ≥2 consumer thực tế hoặc là capability nền tảng rõ ràng; không phụ thuộc model nghiệp vụ; contract ổn định và test độc lập; có owner chịu trách nhiệm versioning/backward compatibility; việc dùng chung **giảm tổng độ phức tạp**, không chỉ chuyển code sang chỗ khác.

**P3 — Default ở library, giá trị môi trường ở deployment.**

```text
Environment / Secret / command line
        ↓ override
application.yml của service
        ↓ override
defaults có tên riêng của library
```

Default resource phải có tên riêng (`vhm-common-defaults.yml`, `vhm-web-defaults.yml`, `vhm-client-defaults.yml`) — không đóng gói nhiều `application.yml` cùng tên trong các JAR vì thứ tự classpath khó dự đoán.

**P4 — Auto-configuration phải có điều kiện.**
Mọi bean hạ tầng dùng điều kiện classpath / property enablement / bean missing. Service không dùng Redis, Thrift hoặc internal HMAC không được fail khởi động chỉ vì default property rỗng. Đây là nguyên tắc then chốt để library dùng chung không biến thành *monolith dùng chung*.

**P5 — Cấu hình local không được rò vào test.**
Job nền và integration phải có công tắc kích hoạt tường minh.

P3, P4, P5 rút ra trực tiếp từ các lỗi runtime ở §2.8.

---

## 4. Cấu trúc mục tiêu

```text
new-structure/
├── libraries/                    # sản phẩm kỹ thuật dùng lại, có vòng đời phát hành riêng
│   ├── pom.xml                   # reactor để build local khi artifact chưa publish
│   ├── vhm-spring-boot-parent/
│   │   └── pom.xml
│   ├── vhm-common/
│   │   ├── src/main/java/vn/vhm/common/
│   │   └── src/main/resources/vhm-common-defaults.yml
│   ├── vhm-web-starter/
│   │   ├── src/main/java/vn/vhm/web/
│   │   └── src/main/resources/vhm-web-defaults.yml
│   └── vhm-client/
│       ├── src/main/java/vn/vhm/client/
│       ├── src/main/thrift/      # IDL nguồn, không đặt trong domain service
│       └── src/main/resources/vhm-client-defaults.yml
└── service/                      # ứng dụng deploy được, sở hữu nghiệp vụ và dữ liệu
    ├── vhm-dossier-core/
    ├── vhm-ocr-ekyc/
    └── vhm-campaign-core/
```

`new-structure` là tên làm việc; khi rollout có thể đổi theo convention của tổ chức, nhưng hai nhóm logical `libraries` / `service` cần giữ nguyên.

### 4.1. `vhm-spring-boot-parent`

Maven parent/convention, **không chứa Java runtime code**.

| Chịu trách nhiệm | Không chịu trách nhiệm |
|---|---|
| Chốt Java và Spring Boot baseline | Chứa business dependency của một service cụ thể |
| Quản version dependency và bản vá CVE thống nhất | Kéo mọi thư viện runtime vào mọi service để POM ngắn |
| Compiler, annotation processor, Surefire, Spring Boot plugin | Ép OCR/worker mang Syncfusion/POI/Thymeleaf nếu không dùng |
| Repository nội bộ/vendor khi thực sự dùng chung | |
| Test baseline | |

POM service có thể ngắn, nhưng "ít dòng" không phải mục tiêu. **Classpath phải đúng và tối thiểu.** Dùng `dependencyManagement` cho version; chỉ cung cấp platform bundle khi tất cả service thực sự cần cùng dependency.

### 4.2. `vhm-common`

```text
vhm-common/src/main/java/vn/vhm/common/
├── annotation/     # annotation kỹ thuật dùng chung
├── config/         # cấu hình nền tảng tái sử dụng
├── persistence/    # base entity, UUIDv7, repository primitive
├── security/       # signing/cipher primitive không phụ thuộc HTTP
└── util/           # utility thuần
```

| Chịu trách nhiệm | Không chịu trách nhiệm |
|---|---|
| `BaseEntity`, `BaseEntityUUID`, UUIDv7 generator, generic repository helper | Dossier status, OCR vendor, notification template, business error code |
| Utility thuần đã có test: file, hash, JSON, string, UUID | Client contract của một upstream cụ thể |
| Security primitive cross-cutting: signing, verification, payload/response cipher | Controller, HTTP response, servlet filter |
| Default kỹ thuật: virtual threads, datasource pool, JPA baseline, Liquibase, structured logging | |
| Kafka/Redisson configuration dùng chung (baseline hiện tại) | |

`logback-spring.xml` canonical đặt tại đây vì logging áp dụng cho cả web service, worker và consumer. Chỉ một bản canonical để tránh classpath chọn nhầm file; service override level/appender qua environment, không copy nguyên file.

**Lưu ý dài hạn:** nếu Kafka/Redis/JPA không còn là baseline của mọi service, tách thành starter tùy chọn thay vì để `vhm-common` phình to (ngưỡng tách ở §11.3).

### 4.3. `vhm-web-starter`

```text
vhm-web-starter/src/main/java/vn/vhm/web/
├── autoconfigure/  # tự cấu hình web có điều kiện
├── controller/     # base/health endpoint
├── dto/            # response, paging, error contract
├── exception/      # exception chung + global handler
├── security/       # HTTP security/filter
└── support/        # web-specific helper
```

| Chịu trách nhiệm | Không chịu trách nhiệm |
|---|---|
| API response envelope, paging contract | Business exception code của dossier/OCR |
| Global exception handler, error payload kỹ thuật | REST client gọi hệ thống ngoài |
| Spring MVC, validation, OpenAPI defaults | JPA entity/repository |
| Security filter chain, internal signature, actor context | Logic notification/assignment/workflow |
| CORS và web defaults | |

Service ném `ApiException` với error code chuẩn kỹ thuật; **error code nghiệp vụ vẫn do service định nghĩa** và map qua contract chung. Web starter không quyết định API nghiệp vụ phải có endpoint nào.

### 4.4. `vhm-client`

```text
vhm-client/src/main/java/vn/vhm/client/
├── autoconfigure/  # đăng ký client/capability có điều kiện
├── config/         # typed properties từng client
├── file/  market/  message/  ocr/  profile/    # capability + DTO contract
└── thrift/         # transport, pool, generated integration
```

| Chịu trách nhiệm | Không chịu trách nhiệm |
|---|---|
| Spring `RestClient` infrastructure dùng chung | Quyết định nghiệp vụ *khi nào* gọi client |
| Typed client: File, OCR, VinBigData, Market, Message Delivery, Profile | Map kết quả upstream thành trạng thái dossier/OCR |
| Request/response DTO đúng contract upstream | Fallback mang ý nghĩa nghiệp vụ |
| Authentication, timeout, error translation, retry policy | DTO API do dossier/OCR public ra ngoài |
| Thrift IDL, generated classes, transport pool, `ProfileClient` wrapper | Phụ thuộc `vhm-web-starter` |

Ví dụ ranh giới: `FileClient.prepareUpload()` thuộc library — quyết định hồ sơ nào được upload loại tài liệu nào thuộc dossier. `ProfileClient` thuộc library — quyết định reviewer nào được phân công thuộc dossier. Message Delivery client gửi request đúng contract — template, nội dung và thời điểm gửi là nghiệp vụ của service.

**Tài liệu đặt cạnh code:** mỗi capability có `GUIDE.md` riêng (`file/`, `market/`, `ocr/`, `message/`, `profile/`) chứa contract, cấu hình, ví dụ, ranh giới và lỗi thường gặp. README root chỉ là mục lục. Guide phải được cập nhật trong **cùng PR** khi đổi property/contract/lifecycle.

### 4.5. Services

**`service/vhm-dossier-core`** — hồ sơ NOXH, note, review case, permission; pipeline/state transition; assignment và SLA/reminder; notification orchestration/outbox; report/export; migration, schema, template.

```text
vhm-dossier-core/
├── src/main/java/.../dossier/
│   ├── controller/  dto/  entity/  repository/  service/
│   ├── workflow/        # trạng thái và chuyển trạng thái
│   ├── notification/    # orchestration thông báo
│   └── report/          # xuất báo cáo/tài liệu
├── src/main/resources/
│   ├── db/  pipelines/  templates/
└── pom.xml              # chỉ dependency chung + dependency riêng thực dùng
```

**`service/vhm-ocr-ekyc`** — OCR/eKYC request lifecycle; provider orchestration và provider-specific adapter; media association và processing job; outbox/event; migration và public API contract.

```text
vhm-ocr-ekyc/
├── src/main/java/.../ocrekyc/
│   ├── controller/  dto/  entity/  repository/  service/
│   ├── provider/    # adapter đặc thù từng nhà cung cấp
│   ├── kafka/       # event nghiệp vụ OCR
│   └── job/         # retry/recovery
├── src/main/resources/db/
└── pom.xml
```

FPT OCR client tiếp tục nằm trong OCR service nếu chỉ OCR dùng và contract gắn chặt provider strategy. **Không đưa mọi outbound call vào `vhm-client` chỉ vì tên class có chữ `Client`.**

**`service/vhm-campaign-core`** — đã kế thừa `vhm-spring-boot-parent` (POM 35 dòng), chưa dùng `vhm-common`/`vhm-web-starter`/`vhm-client`. Áp dụng đầy đủ sau khi pilot OCR hoàn tất.

### 4.6. Dependency rules

```text
vhm-spring-boot-parent          # build/version conventions

vhm-common
   ↑
   ├── vhm-web-starter
   └── vhm-client

service ──> common + web-starter (nếu có HTTP API) + client capability cần dùng
```

Bắt buộc:

- `vhm-common` **không** phụ thuộc `vhm-web-starter` hoặc `vhm-client`.
- `vhm-web-starter` được phụ thuộc `vhm-common`.
- `vhm-client` phụ thuộc `vhm-common`, **không** phụ thuộc `vhm-web-starter`.
- Service phụ thuộc library; library **không** phụ thuộc service.
- Không có dependency cycle.
- Không tạo bean/connection cho capability đang tắt.

Dependency sai chiều `vhm-client → vhm-web-starter` đã được gỡ trong PoC. Nhờ vậy worker/consumer dùng outbound client không bị kéo theo MVC, controller, global handler và web security.

---

## 5. Quy chuẩn cấu hình

### 5.1. Namespace

```yaml
vhm:
  client:
    file: {}
    ocr: {}
    vin-bigdata: {}
    market: {}
    message-delivery: {}
  thrift:
    enabled: false          # mặc định tắt; dossier bật rõ ràng
    services:
      profile: {}
  security: {}              # cipher/signing primitive, namespace trung lập
```

Mỗi client phải có một `@ConfigurationProperties` typed class — không rải rác nhiều `@Value` cho cùng một client. Startup validation fail fast khi capability được enable nhưng thiếu endpoint/credential bắt buộc.

### 5.2. Quy tắc default

- Timeout có default an toàn trong library.
- **Secret không có giá trị mặc định thật.**
- Endpoint localhost chỉ dùng cho local/dev, không làm production fallback âm thầm.
- Feature không dùng phải disable được và không tạo bean/kết nối.
- Service chỉ override khác biệt, không copy toàn bộ default của library.

### 5.3. Bảng quyết định đặt code

Áp dụng theo thứ tự, dừng ở dòng đầu tiên khớp:

| # | Điều kiện | Vị trí |
|---|---|---|
| 1 | Chứa thuật ngữ, trạng thái hoặc quyết định của một domain | Service sở hữu domain |
| 2 | Mô tả contract của upstream, không quyết định business flow | `vhm-client` |
| 3 | Xử lý inbound HTTP/API nhất quán giữa các service | `vhm-web-starter` |
| 4 | Primitive/utility/hạ tầng, không phụ thuộc HTTP và không mang nghiệp vụ | `vhm-common` |
| 5 | Chỉ quản version/plugin/build | `vhm-spring-boot-parent` |
| 6 | Chưa có consumer thứ hai và chưa phải nền tảng bắt buộc | Giữ tại service |

Quy tắc này quan trọng hơn tên package: mục tiêu là **giữ boundary ổn định**, không phải đưa càng nhiều code vào library càng tốt.

### 5.4. Mapping code cũ → cấu trúc mới

| Nhóm code cũ | Vị trí mới | Lý do |
|---|---|---|
| `BaseEntity`, `BaseEntityUUID` | `vhm-common` | Persistence primitive |
| UUIDv7 annotation/generator | `vhm-common` | Dùng chung nhiều service |
| Generic JPA repository/helper | `vhm-common` | Boilerplate persistence |
| File/Hash/JSON/String/UUID utility | `vhm-common` | Utility thuần |
| Payload/response cipher primitive | `vhm-common` | Cross-cutting security |
| Kafka/Redisson baseline | `vhm-common` (hiện tại) | Hạ tầng chung; có thể tách starter sau |
| API envelope, paging, `PageMapper` | `vhm-web-starter` | HTTP/API contract |
| Health/base controller | `vhm-web-starter` | Web concern |
| Global exception handler | `vhm-web-starter` | Chuẩn hóa HTTP error |
| Web security/HMAC filter | `vhm-web-starter` | Servlet/security concern |
| File/OCR/Market/VinBigData client + DTO | `vhm-client` | Integration contract dùng lại |
| Message Delivery client | `vhm-client` | Upstream integration |
| Thrift IDL/runtime/Profile wrapper | `vhm-client` | Transport và contract |
| Dossier workflow/assignment/reminder | `vhm-dossier-core` | Nghiệp vụ |
| OCR request/provider orchestration | `vhm-ocr-ekyc` | Nghiệp vụ |
| `AppErrorCode` theo domain | Giữ tại service | Mã lỗi mang ngữ nghĩa nghiệp vụ |
| `EntityStatus` + converter/serializer | Trả về dossier | Trạng thái permission của domain |
| TTOL roster adapter, role, project mapping, fallback | Trả về dossier | Nghiệp vụ NOXH |
| FPT provider implementation | OCR service, tới khi có consumer thứ hai | Chưa đủ lý do chia sẻ |

---

## 6. Ưu điểm của cấu trúc mới

### 6.1. Ánh xạ nhược điểm → cách xử lý

| # | Nhược điểm cũ | Cách xử lý | Kết quả |
|---|---|---|---|
| 1 | Nền tảng bị xây lặp từng repo | Hạ tầng về library có owner và test | −89 file (dossier), −40 file (OCR) |
| 2 | Không rõ client chính thức | Gom outbound contract vào `vhm-client` | ~66.500 dòng generated code không còn nhân bản |
| 3 | Build/dependency phân tán | Parent quản version/plugin | POM −85…88%; vá CVE tập trung, đo adoption qua version |
| 4 | Cấu hình khó dự đoán | Default library → override service → local profile riêng | Test tái lập, ít phụ thuộc máy dev |
| 5 | Nghiệp vụ lẫn hạ tầng | Service chỉ giữ domain | Reviewer xác định phạm vi ảnh hưởng qua module |
| 6 | Onboarding truyền miệng | Cấu trúc + owner + guide cạnh code | Người mới tự tìm code bằng cấu trúc |
| 7 | Lỗi chỉ lộ khi chạy | Capability enable/disable tường minh | Không mở connection/thread thừa; test ổn định |

### 6.2. So sánh luồng làm việc

| Nhu cầu | Cấu trúc cũ | Cấu trúc mới |
|---|---|---|
| Tạo API mới | Tìm/copy response, exception, validation pattern trong repo | Dùng contract + handler từ `vhm-web-starter` |
| Tạo entity | Tìm base entity/UUID generator đúng phiên bản | Dùng persistence primitive từ `vhm-common` |
| Gọi File/Profile/Market | Search nhiều package/repo, nguy cơ tạo client mới | Tra `vhm-client`, inject typed client |
| Cấu hình Kafka/JPA/Redis | Copy block YAML và config class | Dùng default chung, chỉ override khác biệt |
| Vá lỗi client | Sửa từng bản copy, đồng bộ thủ công | Sửa library, test một contract, nâng version |
| Nâng dependency/CVE | Audit và sửa nhiều POM | Version tập trung; dependency riêng vẫn ở đúng service |
| Debug startup | Xác định property nào thắng trong nhiều nguồn | Namespace thống nhất, capability rõ ràng |

Khác biệt cốt lõi: developer chuyển từ **"tìm một repo để copy"** sang **"dùng một contract có owner và version"**.

### 6.3. Giá trị theo nhóm

**Developer** — không dựng lại logging/security/health/exception mapping/pagination/JPA base/REST infrastructure; thời gian review dồn vào business correctness; giảm import nhầm; test service chỉ kiểm business behavior vì shared concern đã có test tại library.

**Người mới** — mental model 4 phần: `common` = primitive/hạ tầng, `web-starter` = inbound HTTP, `client` = outbound integration, `service` = business logic. Onboarding theo luồng thay vì theo repository: chạy infra local → `.env.local` từ template → build `libraries` hoặc dùng version đã publish → chạy service → tìm use case trong `service` → tìm integration trong `vhm-client`. Typed configuration báo thiếu credential/URL ngay lúc startup với tên property rõ ràng, thay vì null hoặc 401 lúc runtime.

**Quản lý kỹ thuật** — phạm vi sở hữu và review rõ; kiểm soát được version/dependency/CVE; giảm số điểm phải sửa cho một vấn đề nền tảng; **đo adoption qua library version thay vì audit thủ công**; nền móng cho service template và CI/CD chuẩn.

**Vận hành** — logging/error response/cấu hình nhất quán; giảm service mở kết nối tới capability không dùng; điều tra sự cố dễ hơn khi mọi service cùng baseline; rollback bằng cách pin lại version trước.

---

## 7. Kết quả PoC

### 7.1. Số liệu

| Chỉ số | Cũ | Mới | Thay đổi |
|---|---:|---:|---:|
| Dossier Java files | 310 | 221 | −89 file |
| Dossier Java LOC | 26.463 | 21.611 | −18% |
| Dossier POM | 436 | 51 dòng | −88% |
| OCR Java files | 106 | 66 | −40 file |
| OCR Java LOC | 5.695 | 4.206 | −26% |
| OCR POM | 124 | 19 dòng | −85% |

Đo trực tiếp từ source (phạm vi `src/main`) ngày 07/09/2026, đối chiếu lại 08/09/2026.

**Cách đọc số liệu — 3 lưu ý:**

1. Code không "biến mất": phần dùng chung chuyển sang libraries, phần trùng lặp/không cần thiết bị loại bỏ.
2. POM dossier cố ý giữ 51 dòng để khai báo minh bạch dependency riêng (Syncfusion, POI, Thymeleaf, JSON Schema, Gson phục vụ xuất báo cáo/tài liệu). Đưa chúng vào parent chỉ để POM ngắn hơn sẽ làm **mọi** service mang classpath không cần thiết.
3. `vhm-client` có LOC lớn do Java code sinh từ Thrift (~66.500 dòng dưới `vn/vhm/service/agent_profile/`). **Không dùng tổng LOC của module này để đánh giá độ phức tạp do developer tự viết.**

### 7.2. Bằng chứng vận hành

| Hạng mục | Kết quả |
|---|---|
| Libraries reactor | `mvn clean install` — thành công, gồm test từng module |
| OCR/eKYC | Full `mvn verify` — thành công |
| OCR persistence | PostgreSQL Testcontainers + Liquibase — thành công |
| Dossier | `mvn -DskipTests package` — thành công với dependency mới |
| Scheduler isolation | Job OCR tắt đúng trong test profile; test outbox hết tranh chấp |
| Client isolation | OCR không khởi tạo Thrift/Profile pool khi capability tắt |
| Dependency direction | `vhm-client` không còn phụ thuộc `vhm-web-starter` |
| Parent isolation | Dependency export/report đặc thù đã trả về dossier |
| Business leakage audit | Library production source không còn tham chiếu dossier/NOXH/OCR-eKYC/PTT |
| Dossier TTOL tests | `RestPttRosterClientTest`, `PttAutoAssignServiceTest` — thành công sau khi chuyển code về service |

Log lỗi trong một số unit test OCR là dữ liệu test có chủ đích để xác nhận nhánh retry/fallback; Maven kết thúc thành công, không phải lỗi build.

### 7.3. Thay đổi đã thực hiện

**Naming và package** — ba runtime library (`vhm-common`, `vhm-web-starter`, `vhm-client`) + parent convention; root theo `libraries/` và `service/`; API model dùng package `dto`, lỗi web dùng `exception`. Đã loại bỏ phần tạm không cần: correlation ID, tracing package, idempotency abstraction, `TraceHeaders`, `ErrorDescriptor`, `HttpClientSupport` cũ.

**Common** — chuyển utility dùng chung, `BaseEntity`/`BaseEntityUUID`, UUIDv7 annotation/generator, repository foundation, payload/response cipher primitive; gom JPA/Kafka/Redisson/Security baseline; `logback-spring.xml` canonical; default virtual threads, JPA batching, insert/update ordering, structured logging; **loại hard-code JDBC driver** để Spring Boot tự suy ra từ URL, hỗ trợ cả PostgreSQL và H2/Testcontainers.

**Web starter** — chuyển API response, service response, paging/`PageMapper`, generic exception handling, `BaseController`, `HealthController`, web auto-configuration, `ApiException`, `RestControllerExceptionHandler`; SpringDoc/OpenAPI gom về đây. Service giữ business error code.

**Client** — chuyển outbound client của dossier, Thrift IDL/generated/transport pool/factory/Profile wrapper; đổi `AgentProfileClient` → `ProfileClient`; chuẩn hóa Message Delivery contract; dùng Spring `RestClient` thay Apache HttpClient; tạo `ClientResponse` riêng cho transport để client không phụ thuộc API response của inbound web; tách đăng ký REST client properties khỏi `ThriftClientAutoConfiguration`; loại `@Component` thừa ở class đã tạo bằng auto-configuration để tránh duplicate bean khi component scan rộng; Thrift mặc định `vhm.thrift.enabled=false`; file category không còn hard-code `social-housing` mà nhận qua `vhm.client.file.category`; cipher dùng namespace trung lập `vhm.security.*` thay `ocr-ekyc.*`.

**OCR/eKYC** — import library, loại hạ tầng trùng, **không xóa business core**; tách `.env.local` chỉ cho profile `local`; Kafka listener tắt auto-startup trong test nhưng vẫn cung cấp `KafkaTemplate`; ba scheduled component tuân theo `ocr-ekyc.scheduling.enabled` (`OcrOutboxPublisher`, `EkycMediaRetryJob`, `OcrStuckRequestRecoveryJob`); không còn khởi tạo Profile Thrift pool khi không dùng.

**Dossier** — chuyển client/Thrift khỏi domain nhưng giữ orchestration và business mapping; giữ `AppErrorCode` và constant nghiệp vụ; POM kế thừa parent và khai rõ dependency đặc thù; bật Profile/Thrift tường minh qua `vhm.thrift.enabled`.

**Maven** — reactor cho `libraries` để build local khi chưa publish; không dùng Maven "include" kiểu Gradle composite build (local dùng reactor/`mvn install`, CI/CD dùng Maven repository); Kafka loại transitive LZ4 cũ.

---

## 8. Sự cố đã gặp và cách xử lý

### Maven không tìm thấy snapshot library

Artifact chưa publish và chưa cài vào local Maven repository.

```bash
cd new-structure/libraries && mvn clean install
```

`mvn -U` chỉ refresh artifact **đã tồn tại** trên repository, không tạo artifact chưa publish.

### Logging không khởi động

`logback-spring.xml` còn tham chiếu `vn.vinhomes.agent.dossier.core.config.tracing.MaskingPatternLayout` sau khi class đã di chuyển. → Cần một cấu hình logging canonical, và mọi class custom được tham chiếu phải nằm trong dependency runtime.

### PostgreSQL báo role `myuser` không tồn tại

Default trong service không khớp user/password Docker Compose. → Giá trị kết nối local lấy từ `.env.local`/Docker Compose, không hard-code placeholder làm runtime default.

### Kafka listener thiếu factory cũ

Consumer yêu cầu bean `legacyManualCommitKafkaListenerContainerFactory`. → Shared Kafka config cần cung cấp factory/alias tương thích trong giai đoạn migration, hoặc đổi consumer có kiểm soát.

### Integration test OCR không ổn định

Scheduled outbox publisher chạy trong test context và claim row của test case. → Chỉ bật scheduled bean khi `ocr-ekyc.scheduling.enabled=true`; test profile đặt `false`.

### Service không dùng Thrift vẫn tạo pool

Thrift auto-configuration luôn chạy vì default đã có localhost service config. → Thêm capability flag mặc định tắt; chỉ dossier bật khi cần.

---

## 9. Kế hoạch migration

| GĐ | Nội dung | Điều kiện hoàn thành |
|---|---|---|
| **0. ADR và boundary** | Phê duyệt module responsibility và dependency rules; chốt config namespace; chốt danh sách class được phép chuyển; chỉ định owner từng library | ADR được Tech Lead/Architect duyệt |
| **1. Hardening libraries** | Xác nhận parent dependency bằng CI dependency analysis; chuẩn hóa DTO/client còn trùng; phủ đủ conditional auto-configuration contract test; chuẩn hóa documentation/config metadata | Libraries build/test độc lập; dependency graph không cycle; smoke test không tạo capability ngoài nhu cầu |
| **2. Pilot OCR** | Build/package bằng artifact version cố định; chạy unit/integration/contract test; so sánh API response, DB migration, Kafka behavior và logs trước/sau; deploy môi trường test có rollback plan | Không thay đổi business behavior/API ngoài thay đổi đã duyệt |
| **3. Migrate dossier** | Theo capability, **không big-bang** | Mỗi bước có build/test và commit riêng |
| **4. Publish và mở rộng** | Publish release candidate lên Maven repository nội bộ; release notes + migration guide; migrate service tiếp theo | Chỉ tách thêm starter khi có consumer thực tế |

OCR/eKYC được chọn pilot vì phạm vi nhỏ hơn (đã giảm 106 → 66 Java files) và đã qua full `mvn verify` với database thật.

**Thứ tự migrate dossier (GĐ 3):**

```text
1. Parent/POM
2. Common persistence/util
3. Web response/exception/security
4. REST clients
5. Thrift/Profile client
6. Kafka/Redis config
7. Xóa code cũ — chỉ sau khi test và search xác nhận không còn consumer
```

---

## 10. Kiểm thử và đảm bảo không đổi hành vi

**Build gate** — `mvn clean verify` cho reactor libraries và cho từng service; dependency convergence và duplicate-class check; static analysis; secret scan.

**Contract gate** — snapshot/contract test API response và error response; OpenAPI diff; test client request header, authentication và serialization; test Kafka topic/key/payload/ack-mode; test Thrift method/IDL compatibility.

**Runtime gate** — context startup theo local/test/staging profile; database/Liquibase validation; Redis/Kafka health và graceful shutdown; log format/masking verification; **không xuất hiện bean/client không thuộc capability của service**.

**Auto-configuration contract test** — mỗi starter cần tối thiểu: default context khởi động khi capability tắt; bean được tạo khi property hợp lệ; fail-fast message khi capability bật nhưng thiếu config; consumer override được bean/default; không có resource collision; không tạo connection ngoài ý muốn trong test context.

**Rollback** — service pin version library, **không dùng version range**; giữ release trước trong Maven repository; breaking change phải có major version hoặc migration window; database changeset đã chạy không được sửa checksum, phải tạo changeset mới.

---

## 11. Việc còn mở

### 11.1. Ưu tiên cao

- [ ] Thiết lập Maven repository/publishing pipeline, release version, changelog và rollback flow.
- [ ] Thêm auto-configuration contract test theo checklist §10.
- [ ] Secret scan; chỉ giữ `.env.example`, không publish credential thật hoặc lịch sử chứa secret.
- [ ] Chạy full `mvn verify` cho dossier, không chỉ compile/package.
- [ ] Bổ sung Maven Wrapper cho OCR repository hoặc thống nhất lại convention build (hiện verification dùng system Maven trong khi convention cũ yêu cầu `./mvnw -B verify`).
- [ ] Thay `LibraryDefaultsEnvironmentPostProcessor` — đang cảnh báo deprecated trên Spring Boot 4.1, cần cơ chế được hỗ trợ trước khi API bị loại bỏ.

### 11.2. Boundary cần hoàn thiện

- Chỉ đưa TTOL trở lại `vhm-client` nếu có contract tổng quát được ≥2 consumer dùng, và không chứa endpoint/role/project mapping/fallback NOXH.
- `MessageDeliveryClient`, `OcrClient`, `FileManagementClient` hiện là declarative interface — cần proxy bean auto-configuration hoàn chỉnh trước khi quảng bá là inject trực tiếp được ở mọi consumer.
- Rà soát hai nhóm File DTO/client có contract gần nhau. **Không xóa chỉ vì trùng tên**; xác nhận upstream contract rồi mới chuẩn hóa package/capability.
- Chuẩn hóa DTO convention trong `vhm-client` — hiện có DTO ở cả `vn.vhm.client.dto.*` và `vn.vhm.client.<capability>.dto.*`. Ưu tiên colocate theo capability:

```text
vn.vhm.client.file        vn.vhm.client.file.dto
vn.vhm.client.market      vn.vhm.client.market.dto
```

- Quyết định packaging cho Thrift: giữ generated code trong `vhm-client` hay tách `vhm-profile-client` riêng khi số IDL tăng hoặc release cadence khác nhau.

### 11.3. Ngưỡng tách optional starter

`vhm-common` hiện chứa JPA, Kafka và Redisson — lựa chọn thực dụng cho hai service đầu, nhưng không phù hợp nếu có stateless web service hoặc worker không dùng DB/Redis. Tách `vhm-persistence-starter` / `vhm-kafka-starter` / `vhm-redis-starter` khi xuất hiện **một trong ba** dấu hiệu:

1. Có service đầu tiên không dùng capability nhưng vẫn bị kéo dependency/bean.
2. Startup tạo kết nối ngoài không cần thiết.
3. Dependency/CVE surface tăng đáng kể.

**Không tách sớm** thành nhiều module nhỏ khi chưa có nhu cầu thực tế.

### 11.4. Tài liệu và governance

- Mỗi client guide cập nhật cùng PR thay đổi contract/property.
- Bổ sung configuration metadata để IDE autocomplete property.
- Chốt owner cho parent/common/web/client và owner từng upstream contract.
- Chốt ADR về boundary, version compatibility và deprecation window.

---

## 12. Governance và Go/No-Go

### 12.1. Ownership

| Vai trò | Phạm vi |
|---|---|
| Platform / backend enablement | `vhm-spring-boot-parent`, `vhm-common`, `vhm-web-starter` |
| Integration owner | Từng client contract trong `vhm-client` |
| Domain team | Business service và business error code |

### 12.2. Pull request checklist cho thay đổi shared library

1. Có bao nhiêu consumer?
2. Có mang business assumption không?
3. Có backward compatible không?
4. Default có an toàn không?
5. Consumer có override/disable được không?
6. Test nào chứng minh behavior?

### 12.3. Version rule

- **Patch** — bug fix backward compatible.
- **Minor** — capability mới, default không phá vỡ consumer.
- **Major** — đổi package, property prefix, contract hoặc behavior breaking.
- **Không dùng `SNAPSHOT` trong production deployment.**

### 12.4. Điều kiện Go/No-Go cho rollout rộng

Chỉ chuyển từ PoC sang chuẩn áp dụng rộng khi **tất cả** đạt:

- [ ] Libraries publish từ CI lên Maven repository nội bộ bằng version bất biến.
- [ ] Không có dependency cycle, duplicate class hoặc dependency sai chiều.
- [ ] Mỗi capability tắt được mà không tạo bean/kết nối ngoài nhu cầu.
- [ ] OCR pilot vượt unit, integration, API contract, Kafka và migration check.
- [ ] Rollback được bằng cách pin lại artifact version trước.
- [ ] Có owner, changelog, migration guide và SLA xử lý lỗi library.
- [ ] Không có credential thật trong source/history được phát hành.

Nếu một điều kiện chưa đạt: tiếp tục dùng trong phạm vi pilot, **không ép service khác migrate**.

### 12.5. Rủi ro và kiểm soát

| Rủi ro | Tác động | Kiểm soát |
|---|---|---|
| Shared library thành "god library" | Coupling toàn hệ thống | Boundary §5.3, owner, review checklist §12.2, optional starter §11.3 |
| Một release lỗi ảnh hưởng nhiều service | Blast radius lớn | Version pin, RC, contract test, canary, rollback |
| Breaking config/package | Service không khởi động | Migration guide, deprecation window, major version |
| Parent kéo dependency thừa | Classpath/CVE/startup tăng | Dependency analysis; parent chỉ quản convention/version |
| Business logic lọt vào common | Domain coupling | Rule P1/P2, architecture review, audit định kỳ |
| Local reactor khác CI artifact | Lỗi chỉ xuất hiện khi deploy | CI build từ artifact đã publish, reproducible version |
| Secret nằm trong repository | Security incident | `.env.example`, secret manager, gitleaks, history cleanup |

### 12.6. Chỉ số đánh giá sau rollout

Theo dõi 2–3 sprint, dùng dữ liệu thực tế để tính ROI thay vì cam kết trước một con số tiết kiệm:

- Thời gian tạo và chạy được một service mới.
- Thời gian onboarding đến PR nghiệp vụ đầu tiên.
- Số class boilerplate bị copy giữa repository.
- Số version Spring/dependency khác nhau trong hệ thống.
- Số lỗi do config prefix/default sai.
- **Số nơi phải sửa cho một CVE hoặc bug client.**
- Thời gian review trung bình của PR hạ tầng vs PR nghiệp vụ.
- Startup time, memory footprint và tổng dependency của từng service.

Mục tiêu không phải giảm LOC. Thành công là **giảm thời gian tìm kiếm, giảm quyết định lặp, giảm drift và tăng khả năng thay đổi an toàn**.

---

## 13. Nội dung đề nghị phê duyệt

Phê duyệt ở giai đoạn này là phê duyệt **boundary, ownership và pilot** — không phải phê duyệt một lần để thay đổi đồng loạt. Thay đổi business API, schema hoặc behavior vẫn qua quy trình review riêng của từng domain.

1. Mô hình `libraries/` + `service/` làm cấu trúc chuẩn cho Java service thuộc phạm vi NOXH.
2. Bốn artifact nền tảng: `vhm-spring-boot-parent`, `vhm-common`, `vhm-web-starter`, `vhm-client`.
3. Nguyên tắc service chỉ giữ business logic và adapter đặc thù; library không chứa business rule.
4. Cho phép hoàn tất hardening trước khi publish chính thức.
5. Chọn OCR/eKYC làm pilot, sau đó migrate dossier theo từng capability.
6. Thiết lập owner, versioning và Maven repository nội bộ cho libraries.
7. Không rollout bắt buộc cho service khác cho tới khi hoàn thành §11.1 và §12.4.

| Hạng mục | Deliverable | Chủ trì | Xác nhận |
|---|---|---|---|
| Kiến trúc | ADR module boundary + dependency rules | Tech Lead/Architect | Backend leads |
| Libraries | Artifact, test, changelog, config metadata | Platform/backend enablement | Consumer teams |
| CI/CD | Build, publish, version pin, rollback | DevOps + library owner | Tech Lead |
| OCR pilot | Regression/contract/runtime evidence | OCR team | QA + domain owner |
| Dossier migration | Migration theo capability | Dossier team | QA + domain owner |
| Security | Secret scan + dependency/CVE gate | Security/DevSecOps | Release owner |

---

## 14. Kết luận

Cấu trúc cũ không sai trong bối cảnh một service phát triển độc lập, nhưng không còn hiệu quả khi nhiều service cùng dùng Spring Boot, cùng gọi các upstream và cùng phải đáp ứng tiêu chuẩn security/observability.

Chi phí lớn nhất không nằm ở số dòng code, mà ở **thời gian tìm kiếm, phân biệt nhiều implementation gần giống nhau, đồng bộ bản vá và hướng dẫn người mới** — đều là chi phí tăng theo số biến thể, không theo quy mô code.

Cấu trúc mới tạo boundary rõ: build convention, common primitive, inbound web standard, outbound client, business service. PoC cho thấy service nhỏ hơn đáng kể (−18…26% LOC), POM gần như chỉ còn metadata (−85…88%), ~66.500 dòng generated code không còn phải nhân bản, client dễ tìm, cấu hình có một nguồn mặc định — và vẫn build/test thành công.

Để đủ chất lượng triển khai rộng còn cần hoàn thiện dependency isolation, auto-configuration conditions, versioning/publishing và secret management. Với rollout theo giai đoạn và quality gate ở §12.4, lợi ích dài hạn lớn hơn chi phí migration, đồng thời rủi ro kiểm soát và rollback được.

---

## Phụ lục A — Quy tắc khi tiếp tục refactor

1. Không xóa business core khỏi service chỉ để giảm LOC.
2. Không chuyển class vào common nếu class mang domain assumption.
3. Không thêm dependency service-specific vào parent để làm POM ngắn.
4. Không để client phụ thuộc web starter hoặc service.
5. Không tạo bean/connection cho capability đang tắt.
6. Không đổi package/property prefix mà không cập nhật consumer, test, guide và migration note.
7. Sau thay đổi production code phải chạy verify phù hợp; ghi kết quả mới vào tài liệu này.
8. Giữ code cũ làm nguồn đối chiếu cho tới khi migration được kiểm chứng; không xóa repository cũ trong giai đoạn PoC.

## Phụ lục B — Lệnh thường dùng

```bash
# Build toàn bộ libraries (bắt buộc khi artifact chưa publish)
cd new-structure/libraries && mvn clean install

# Verify một service
cd new-structure/service/vhm-ocr-ekyc && mvn -B verify

# Kiểm tra dependency thừa/thiếu
mvn dependency:analyze
mvn dependency:tree
```

## Phụ lục C — Tài liệu module

| Vị trí | Nội dung |
|---|---|
| `libraries/vhm-client/README.md` | Mục lục và quy tắc chung của client library |
| `libraries/vhm-client/src/main/java/vn/vhm/client/*/GUIDE.md` | Guide từng capability: `file`, `market`, `ocr`, `message`, `profile` |

Guide của capability phải được cập nhật trong **cùng pull request** khi thay đổi property, contract hoặc lifecycle của client đó.

## Phụ lục D — Codebase đối chiếu

| Repository | Đường dẫn |
|---|---|
| Dossier (cũ) | `/home/huynv106/Documents/o2o/noxh/social-housing/vhm-dossier-core` |
| OCR/eKYC (cũ) | `/home/huynv106/Documents/o2o/noxh/orc-ekyc/vhm-ocr-ekyc` |
| Cấu trúc mới | `/home/huynv106/Documents/o2o/noxh/new-structure` |
