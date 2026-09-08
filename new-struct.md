# NOXH Java Platform

> Nền tảng dùng chung cho các service Java 25 / Spring Boot 4 của Vinhomes: cấu trúc module, ranh
> giới ownership và cách quyết định một thành phần nằm ở library hay service.

## 1. Vấn đề cần giải quyết

Cấu trúc cũ (`ocr-ekyc`, `vhm-dossier-core`, `vhm-campaign-core`) đặt code nghiệp vụ và code nền
tảng chung một namespace service. Khi service khác cần cùng capability, cách nhanh nhất là copy;
mỗi bản copy sau đó có lifecycle riêng và trở thành một biến thể mới.

> **Nguyên nhân gốc:** không có ranh giới ownership giữa platform, inbound web, outbound
> integration và business domain.

Hệ quả điển hình: base entity/UUID generator/API response/exception handler/Kafka/Redis/security
filter tồn tại nhiều bản trùng tên nhưng **khác hành vi** (status mapping, serializer, retry, path
matcher…); outbound client và DTO upstream bị khóa trong service đầu tiên dùng chúng; Kafka/Redis có
nhiều namespace cấu hình nên không rõ bean nào thực sự bind; mỗi POM tự khai báo version nên nâng
Spring Boot hay vá CVE phải sửa từng repository. Sai lệch loại này thường không gây compile error —
chỉ lộ ra khi tích hợp hoặc vận hành.

Chi phí tăng theo **số implementation phải duy trì**, không theo số dòng bị copy: một bản vá nền
tảng phải phân tích mọi biến thể → sửa từng repo → test từng bản → release nhiều lịch → theo dõi
service chưa cập nhật.

| Điểm yếu cũ | Yêu cầu mới | Thành phần chịu trách nhiệm |
|---|---|---|
| Version/plugin khác nhau | Một build baseline có version | `vhm-spring-boot-parent` |
| Primitive, JPA, Kafka, Redis bị copy | Một implementation hạ tầng thấp nhất | `vhm-common` |
| Response, exception, security phân mảnh | Một inbound HTTP contract cấu hình được | `vhm-web-starter` |
| Client/DTO rải rác trong service | Typed client nhóm theo upstream | `vhm-client` |
| Business trộn infrastructure | Service chỉ giữ hành vi domain | Từng service |

Mục tiêu **không** phải đưa càng nhiều code vào common, mà là giảm số biến thể phải bảo trì: mỗi
capability có một contract có owner và version. Đổi lại, library phải giữ backward compatibility
hoặc công bố breaking change rõ ràng, có test tại library và compatibility test tại consumer, và
service phải pin version bất biến trong production.

## 2. Cấu trúc và quy tắc phụ thuộc

```text
new-structure/
├── libraries/
│   ├── vhm-spring-boot-parent/  # Maven/build baseline
│   ├── vhm-common/              # primitive và infrastructure dùng chung
│   ├── vhm-web-starter/         # inbound HTTP contract và web security
│   └── vhm-client/              # outbound client và upstream contract
└── service/
    ├── vhm-dossier-core/  ├── vhm-ocr-ekyc/  └── vhm-campaign-core/
```

```text
service → vhm-web-starter, vhm-client → vhm-common
```

1. Library không phụ thuộc ngược vào service.
2. `vhm-common` không phụ thuộc `vhm-web-starter` hoặc `vhm-client`.
3. `vhm-client` không phụ thuộc `vhm-web-starter`.
4. Business entity, error code, trạng thái và quyết định nghiệp vụ luôn ở service.
5. Capability có kết nối ra ngoài phải bật/tắt được; capability không dùng không được làm service
   fail startup.
6. Khác biệt giữa các service biểu diễn bằng YAML/property, không copy config class.

## 3. Vai trò từng library

### 3.1. `vhm-spring-boot-parent`

`packaging=pom`, không có Java source. Sở hữu: Java 25 + Spring Boot 4, dependency/version
management và bản vá tập trung, annotation processor (Lombok, MapStruct, Hibernate), Maven
Compiler/Surefire/Spring Boot plugin, test dependency chuẩn, Maven repository và version của các
VHM library.

Service ở repository riêng phải resolve parent đã publish:

```xml
<parent>
    <groupId>vn.vinhomes.platform</groupId>
    <artifactId>vhm-spring-boot-parent</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <relativePath/>
</parent>
```

`<relativePath/>` rỗng ngăn Maven tìm đường dẫn local không tồn tại trong CI/CD.

Không thuộc parent: Java class, application config, dependency/plugin của một domain, secret.

**Trạng thái hiện tại:** đây là *opinionated application parent* — nó khai báo trực tiếp
`vhm-common`, `vhm-web-starter`, `vhm-client`. Phù hợp baseline HTTP service, nhưng worker không
dùng web vẫn nhận web classpath (xem mục 9).

### 3.2. `vhm-common`

Lớp nền thấp nhất: primitive và infrastructure không gắn với inbound HTTP, một upstream cụ thể hay
một domain cụ thể.

| Package | Trách nhiệm |
|---|---|
| `common.annotation` | Chuẩn sinh định danh UUIDv7 cho entity |
| `common.entity` | `BaseEntity`, `BaseEntityUUID`, `AuditedEntity*` — field persistence và audit |
| `common.repository` | Base repository, condition/assignment builder, `@EnablePersistence`; không chứa query nghiệp vụ |
| `common.config` | Auto-configure cache, Kafka (kèm legacy), Jackson-Kafka, Redisson từ property |
| `common.crypto` | Canonical request, HMAC signer/verifier, cipher, signature header |
| `common.util` | File, hash, HTTP, JSON, validation, phone, masking, string, UUID |

Tên `crypto` là có chủ ý: đây là công cụ ký/mã hóa, không phải authentication/authorization (do
`vhm-web-starter` sở hữu).

`CommonAutoConfiguration` là entry point; default nằm ở `vhm-common-defaults.yml` (virtual threads,
datasource/Hikari/JPA/Liquibase, Kafka, Redis/Redisson, structured logging) và được
`LibraryDefaultsEnvironmentPostProcessor` nạp ở ưu tiên thấp để service/environment luôn override
được. Cipher chỉ tạo khi có key `vhm.crypto.*` tương ứng.

Module cung cấp baseline dependency cho validation, actuator, JPA/Liquibase/PostgreSQL,
cache/Caffeine, Kafka, Redisson, HttpClient 5, Apache POI. Việc dependency nằm ở common chỉ chuẩn
hóa implementation/version, không phải lý do đưa mọi code dùng dependency đó vào common.

Không đặt ở đây: controller/filter/API response/HTTP status mapping; client và DTO của một upstream;
business error code, domain enum/entity, query nghiệp vụ; workflow, scheduler, Kafka listener
nghiệp vụ; secret hoặc URL của một môi trường.

### 3.3. `vhm-web-starter`

Chuẩn hóa **inbound HTTP boundary** cho Spring MVC.

| Package | Trách nhiệm |
|---|---|
| `web.controller` | `BaseController`, `HttpResponse`, health endpoint |
| `web.dto` | `ApiResponse`, metadata, paging, `ServiceResponse` |
| `web.exception` | Generic exception, error payload, global handler |
| `web.config` | `SecurityConfig`, timezone-aware Jackson |
| `web.security.realm` | Realm properties, CIDR filter |
| `web.security.internal` | HMAC filter, replay guard, actor context |
| `web.security.upstream` | BFF registry/filter, current actor, data scope |
| `web.timezone` / `web.util` | Request timezone context; `IpUtil` |

**Một `SecurityConfig`, khác biệt đi qua YAML.** Service không tạo `SecurityConfig` song song; realm,
path, CIDR, internal signature, actor context và upstream BFF khai báo qua `security.*` và
`vhm.web.*`. `WebAutoConfiguration` cung cấp OpenAPI, CORS, global exception handler, timezone và
web bean mặc định; `@ConditionalOnMissingBean` cho phép thay thế có kiểm soát,
`@ConditionalOnProperty` chỉ bật khi YAML yêu cầu. Override bean là escape hatch cho mechanism thực
sự khác, không phải cách cấu hình thông thường.

**Ranh giới exception:** starter sở hữu *cơ chế* và generic HTTP exception (bad request, forbidden,
invalid state, not found); service sở hữu *vocabulary lỗi nghiệp vụ* (`CampaignErrorCode`,
`DossierApprovalException`) dù handler của starter serialize response.

Không đặt ở đây: outbound client, JPA entity/repository, phân quyền gắn workflow domain, DTO của
một API nghiệp vụ, credential thật.

### 3.4. `vhm-client`

Outbound integration layer dùng chung: đóng gói kết nối, authentication, timeout, transport và DTO
theo contract upstream. Service inject typed client, quyết định khi nào gọi và map kết quả vào
domain.

Client, properties, exception, DTO của cùng upstream đặt cạnh nhau:

```text
vn.vinhomes.client/
├── file/ iam/ incom/ market/ message/ profile/   # mỗi capability tự chứa client + config + DTO
├── ocr/{dto,provider/dto,vinbigdata}
└── thrift/                                        # factory, validator, pool, transport config
```

Package gốc `vn.vinhomes.client` chỉ chứa infrastructure chung: `RestClientSupport`/`RestClients`,
basic authentication, `ClientException`, `ClientAutoConfiguration`. Không tạo lại `client.config`
hoặc `client.dto` gộp nhiều upstream. Generated Thrift contract nằm dưới
`vn.vinhomes.service.agent_profile` — generated source, không chỉnh tay.

Cách dùng: cấu hình property group đúng (credential từ Vault/environment) → inject typed client →
gọi operation → map response/exception sang model và outcome của domain.

```yaml
vhm:
  client:
    file:
      base-url: ${FILE_CLIENT_BASE_URL}
      secret-key: ${FILE_CLIENT_SECRET_KEY}
      connect-timeout: 3s
      read-timeout: 20s
```

URL có thể có default local an toàn; username, password, HMAC key và API key thật không commit.

**Điều kiện để một client vào library:** contract upstream ổn định; có ít nhất hai consumer thực tế
hoặc là platform capability đã nhận ownership; API không lộ model của service đầu tiên; timeout,
authentication và error semantics chuẩn hóa được. Client chỉ phục vụ một domain thì ở lại service —
hậu tố `Client` không tự làm nó shared code.

Không đặt ở đây: quyết định khi nào gửi notification/duyệt dossier/chạy campaign, fallback làm thay
đổi business outcome, domain entity, controller DTO, web security filter.

## 4. Service sở hữu gì

```text
src/main/java/vn/vinhomes/<domain>/
├── controller/ dto/ model|entity/ repository/ service/ mapper/
├── event|kafka/   # event schema, listener, publisher của domain
├── scheduler/     # job nghiệp vụ
├── client/        # adapter chỉ service này dùng
├── config/        # extension thật sự đặc thù
└── exception/     # business error code/exception
```

Cùng với: Liquibase changelog của schema mình; topic name, consumer group, concurrency, payload;
endpoint mapping theo môi trường; permission vocabulary, feature flag, rate limit; test use
case/migration/integration; Dockerfile và deployment manifest.

Platform cung cấp machinery — Kafka/Redis, security mechanism, client. Service sở hữu topic và cách
xử lý message, permission và policy, quyết định gọi client và xử lý kết quả.

## 5. Bảng quyết định đặt code

Áp dụng theo thứ tự, dừng ở điều kiện đầu tiên khớp:

| Câu hỏi | Vị trí |
|---|---|
| Có thuật ngữ, trạng thái hoặc quyết định của một domain? | Service sở hữu domain |
| Mô tả contract/transport của upstream dùng chung? | `vhm-client/<capability>` |
| Xử lý inbound HTTP giống nhau giữa các service? | `vhm-web-starter` |
| Primitive kỹ thuật, không phụ thuộc HTTP, không có business rule? | `vhm-common` |
| Chỉ quản version, plugin hoặc build baseline? | `vhm-spring-boot-parent` |
| Chưa có consumer thứ hai và không phải platform capability? | Giữ tại service |

Ví dụ: `UUIDv7Generator`, Kafka producer factory, HMAC signer → `vhm-common`;
`RestControllerExceptionHandler`, servlet HMAC authentication filter → `vhm-web-starter`;
`FileClient` và File DTO → `vhm-client/file`; `CampaignStatus`, `CampaignErrorCode`, notification
topic/listener, quyết định loại tài liệu dossier cần upload → service tương ứng.

Trùng tên ở hai service chưa đủ điều kiện để share; chỉ gom khi semantics, lifecycle và owner thực
sự giống nhau.

## 6. Cấu hình và ownership

```text
Vault / K8s Secret / environment variable   →  ưu tiên cao nhất
application(-<profile>).yml                 →  service/environment
vhm-*-defaults.yml trong library            →  default thấp nhất
```

- Library default phải an toàn và không chứa secret; service chỉ override phần khác biệt.
- Secret dùng `${ENV_NAME}`, do Vault/deployment inject.
- Feature tùy chọn phải có `enabled` hoặc điều kiện bean rõ ràng.
- Local profile (port/database/schema local) nằm ở service.
- Không sửa changeset Liquibase đã chạy; tạo changeset mới. Tắt Liquibase local không sửa được
  checksum ở môi trường dùng chung.

| Namespace | Owner |
|---|---|
| `spring.datasource`, `spring.jpa`, `spring.liquibase` | common cung cấp baseline; service cung cấp URL/schema |
| `kafka.*`, `redisson.config.*`, `vhm.crypto.*` | `vhm-common` |
| `security.*`, `vhm.web.*`, `vhm.cors.*`, `springdoc.*` | `vhm-web-starter` |
| `vhm.client.<capability>.*`, Thrift properties | `vhm-client` |
| `campaign.*`, `dossier.*`, `segment.*`, … | service tương ứng |

## 7. Thêm bean/capability vào library

Entry point đăng ký qua `AutoConfiguration.imports`: `CommonAutoConfiguration`,
`WebAutoConfiguration`, `ClientAutoConfiguration` và Thrift auto-configuration.

1. `@ConditionalOnClass` nếu capability cần dependency tùy chọn.
2. `@ConditionalOnProperty` nếu capability bật/tắt được.
3. `@ConditionalOnMissingBean` khi service được phép thay implementation.
4. Validate property bắt buộc khi feature bật; lỗi phải nêu đúng tên property thiếu.
5. Không mở connection, pool hoặc thread khi capability tắt.
6. Test cả trạng thái bật và tắt.

Trước khi move code từ service vào library: xác định consumer thực tế và owner; chứng minh API không
lộ business model; chọn library bằng bảng mục 5; thiết kế namespace property và default an toàn;
viết test + JavaDoc; kiểm tra dependency direction và classpath impact; publish version và migrate
từng consumer; chỉ xóa implementation cũ sau khi service build/test thành công.

**JavaDoc cho public library API** phải đủ để dùng mà không đọc implementation: class giải quyết và
không giải quyết việc gì; property bắt buộc/tùy chọn và nguồn secret; cách inject; ví dụ gọi ngắn;
điều kiện auto-configuration; lỗi quan trọng, thread-safety, lifecycle. Tài liệu này giải thích
boundary — JavaDoc giải thích cách dùng từng class; không duy trì `GUIDE.md` lặp lại API.

## 8. Publish giữa các Git repository

Library và service ở repository độc lập:

1. Build và test platform libraries.
2. Publish parent POM và JAR lên Maven repository nội bộ.
3. Service pin một platform version đã publish, `<relativePath/>` rỗng.
4. CI service nhận credential đọc registry qua `settings.xml` hoặc CI variable.
5. Build service trong môi trường sạch.
6. Nâng version qua merge request có changelog và compatibility test.

`mvn install` chỉ dành cho local development. Build chỉ chạy được trên máy đã install local mà chưa
publish artifact là build không tái lập. Release không dùng version mutable; snapshot cần snapshot
policy rõ ràng.

## 9. Kết quả đo và technical debt

| Chỉ số | Trước | Sau | Thay đổi |
|---|---:|---:|---:|
| Dossier Java files (`src/main`) | 310 | 221 | −89 |
| Dossier Java LOC | 26.463 | 21.611 | −18% |
| Dossier POM | 436 dòng | 51 dòng | −88% |
| OCR/eKYC Java files | 106 | 66 | −40 |
| OCR/eKYC Java LOC | 5.695 | 4.206 | −26% |
| OCR/eKYC POM | 124 dòng | 19 dòng | −85% |
| Thrift generated code trong mỗi service dùng Profile | ~66.500 dòng | 0 | chuyển về `vhm-client` |

Số liệu là snapshot tại thời điểm refactor, dùng để chứng minh xu hướng giảm duplication chứ không
phải chỉ tiêu chất lượng. LOC giảm là kết quả của việc bỏ duplication; `vhm-client` vẫn có LOC lớn
do generated Thrift.

Đã kiểm chứng: libraries reactor build/install thành công; service resolve parent từ Maven
repository mà không cần relative path; `vhm-client` không phụ thuộc `vhm-web-starter`; shared code
thống nhất dưới `vn.vinhomes.*`; OCR/eKYC load local profile và datasource.

Debt còn mở:

1. **Parent kéo mọi runtime library** — cân nhắc BOM + opt-in starter khi có consumer worker thực tế.
2. **`HealthController` vẫn trong `vhm-web-starter`** — nếu Actuator là health contract duy nhất,
   cần xóa class và bean đăng ký.
3. **`logback-spring.xml` vẫn trong `vhm-common`** — nếu logging chỉ cấu hình bằng YAML/deployment,
   bỏ resource và test lại precedence.
4. **`LegacyKafkaConfig` còn tồn tại** — cần deadline migrate consumer rồi xóa legacy path.
5. **Classpath common khá rộng** (JPA, Liquibase, Kafka, Redis, POI, HTTP client) — nếu service nhẹ
   bị ảnh hưởng, tách starter theo capability.

## 10. Definition of Done cho service đã migrate

- Parent dùng version đã publish, `<relativePath/>` rỗng.
- Không copy base entity, UUID generator, Kafka/Redis config, web exception handler hay shared
  security config.
- Client/DTO upstream dùng chung đến từ `vhm-client`, nhóm theo capability.
- Service chỉ giữ business exception, model, repository, workflow và adapter đặc thù.
- Cấu hình mở rộng đi qua YAML/property; secret từ Vault/environment.
- Package shared dùng `vn.vinhomes.*`; public library API có JavaDoc.
- `mvn verify` chạy thành công trong môi trường sạch.

Kết quả mong muốn: developer chuyển từ **“tìm một repository để copy”** sang **“dùng một contract có
owner, version và hướng dẫn sử dụng”**.
