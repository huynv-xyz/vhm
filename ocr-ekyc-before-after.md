# VHM Common

Thư viện nền dùng chung cho các backend service Java 25 và Spring Boot 4 của Vinhomes. Một JAR cung
cấp các contract và hạ tầng kỹ thuật dùng xuyên service; business logic vẫn thuộc service sở hữu
nghiệp vụ.

## Version và package

| Dòng API | Package | Trạng thái sử dụng                         |
|---|---|--------------------------------------------|
| 1.0.2 | `qtvhbds.common.*` | API đang được các service hiện hữu sử dụng |
| 1.0.3+ | `vn.vinhomes.common.*` | API mới bổ sung từ version 1.0.3           |

Hai dòng API được đóng gói trong cùng artifact. Trong release `1.x`, không rename, move hoặc xóa
class thuộc `qtvhbds.common.*`. Service có thể nâng dependency trước rồi chuyển từng phần sang
`vn.vinhomes.common.*`, không cần migrate toàn bộ cùng lúc.

## Cài đặt

### Quick Start Checklist

1. Thêm dependency `vhm-common`; không khai báo lại các starter common đã cung cấp.
2. Không copy `vhm-common-defaults.yml`; baseline được tự động nạp từ JAR.
3. Chỉ bật capability service thực sự dùng và truyền endpoint/credential qua ConfigMap/Vault.

### Reference implementation

Service OCR eKYC đã áp dụng API `vn.vinhomes.common.*`, auto-configuration và shared defaults tại
[Merge Request !52](https://gitlab.vinsmartfuture.tech/vsf-qtvhbds/vinhomes/agent/ocr-ekyc/-/merge_requests/52).
MR này để tham khảo cách nâng dependency, xóa code/config trùng lặp và chuyển dần
sang common. Đây là ví dụ tích hợp toàn thư viện, không phải yêu cầu mọi service phải bật toàn bộ
capability; mỗi service vẫn chỉ bật những phần thực sự sử dụng.

#### 1. Phạm vi so sánh

- Trước: `HEAD` của repository `vhm-ocr-ekyc` trước migration.
- Sau: working tree sau khi tích hợp common library.
- Không tính `.env.local` vì đây là cấu hình máy cá nhân và đã được Git ignore.

#### 2. Số liệu tổng quan

| Chỉ số | Trước | Sau | Thay đổi |
|---|---:|---:|---:|
| Production Java files | 112 | 69 | Giảm 43 file (38,4%) |
| Production Java LOC | 6.099 | 4.449 | Giảm 1.650 dòng (27,1%) |
| Direct Maven dependencies | 17 | 8 | Giảm 9 dependency (52,9%) |
| `application.yml` | 144 dòng | 69 dòng | Giảm 75 dòng (52,1%) |
| Tổng diff | — | 316 dòng thêm, 2.366 dòng xóa | Giảm ròng 2.050 dòng |

Riêng production code, config, POM và Dockerfile:

| Chỉ số | Kết quả |
|---|---:|
| File bị tác động | 80 |
| Dòng thêm | 248 |
| Dòng xóa | 2.156 |
| Thay đổi ròng | Giảm 1.908 dòng |

Các phần bị loại bỏ gồm `43` Java class trùng khỏi source chính, trong đó có `6` utility, `5`
repository base và `9` DTO của File Management. Service cũng bỏ implementation riêng cho UUIDv7,
AES-GCM, API response/exception, Basic Auth, `AuthContextProvider`, Logback masking, JPA
configuration và HTTP client infrastructure. `application.yml` không còn lặp Kafka connection
defaults; service chỉ giữ cấu hình nghiệp vụ và capability cần bật.

#### 3. Kết quả kiểm chứng

- `mvn test-compile` chạy thành công.
- Application khởi động local thành công với PostgreSQL và Kafka.
- `/actuator/health` trả HTTP `200` với trạng thái `UP`.

Các con số trên là kết quả của OCR eKYC, không phải mục tiêu bắt buộc cho mọi service. Mức giảm thực
tế phụ thuộc lượng infrastructure đang bị copy trong từng repository.

### Reference implementation: Dossier Core

`vhm-dossier-core` đang tích hợp common trên branch `feat/huynv106/upgrade-common-library` của
[repository Dossier Core](https://gitlab.vinsmartfuture.tech/vsf-qtvhbds/vinhomes/agent/dossier-services/vhm-dossier-core).
Đây là ví dụ cho service lớn đã chạy production, có JPA/Liquibase, Redis, primary và named Kafka
cluster, HTTP clients, Thrift, Basic Auth và security chain riêng.

#### 1. Phạm vi so sánh

- Trước: `origin/staging` tại thời điểm bắt đầu tích hợp.
- Sau: working tree trên branch tích hợp common.
- Không tính `.env.local` vào artifact phát hành; file này chỉ phục vụ máy local và phải được Git
  ignore.
- Branch còn có refactor service implementation ngoài phạm vi common. Vì vậy số liệu tổng thể bên
  dưới mô tả toàn branch; không quy toàn bộ phần giảm dòng cho common.

#### 2. Số liệu tổng quan

| Chỉ số | Trước | Sau | Thay đổi |
|---|---:|---:|---:|
| Production Java files | 316 | 229 | Giảm 87 file (27,5%) |
| Production Java LOC | 27.358 | 22.715 | Giảm 4.643 dòng (17,0%) |
| Direct Maven dependencies | 36 | 12 | Giảm 24 dependency (66,7%) |
| Cấu hình application | 631 dòng trong 4 file properties | 135 dòng trong 1 file YAML | Giảm 496 dòng (78,6%) |
| Tổng diff của branch | — | 6.055 dòng thêm, 13.256 dòng xóa trên 261 file | Giảm ròng 7.201 dòng |

Phần giảm trực tiếp nhờ common gồm:

- Bỏ các implementation HTTP client riêng cho File, Private File, Market, Message Delivery và
  VinBigdata; service inject contract do common cung cấp.
- Bỏ toàn bộ Thrift transport pool/factory/config và generated contract Profile/Personal Contact
  khỏi service; dùng facade của common.
- Bỏ Kafka primary config, Redisson config, OpenAPI config, Basic Auth chain, health controller,
  tracing/logback infrastructure và các utility trùng.
- Bỏ repository base, pagination DTO, API response envelope, UUIDv7 generator và các File
  Management DTO trùng.
- Gom bốn file cấu hình môi trường thành một `application.yml`; baseline Spring/JPA/Kafka,
  Liquibase, observability, HTTP transport và Redisson defaults do common sở hữu. Service chỉ giữ
  topic, named Kafka cluster, internal HMAC security và cấu hình nghiệp vụ dossier.

Các phần vẫn thuộc Dossier Core gồm JSON Schema validation, Syncfusion/POI export, Thymeleaf email,
TTOL roster, workflow/reminder/outbox và internal HMAC authorization. Common không hấp thụ những
thành phần này chỉ để giảm số dòng.

#### 3. Kết quả kiểm chứng

- Common `1.0.3` compile và publish Maven local thành công.
- Production source của Dossier Core chạy `mvn compile` thành công trên working tree sạch không dùng
  generated Thrift cũ.
- Liquibase auto-configuration chạy trước Hibernate; database local được migrate tới changeset
  `V20260909001__dossier_inquiry_id.sql` mà không sửa migration đã chạy trên staging/production.
- PostgreSQL dialect, managed `RedissonClient`, Jackson 2 compatibility mapper, observed
  `RestClient` và hai Thrift facade được tạo từ đúng ownership giữa common và service.
- Reminder Scanner có switch riêng để local không gọi Profile MW khi không có network nội bộ;
  staging/production vẫn mặc định bật.

### POM tối thiểu của service

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.2</version>
        <relativePath/>
    </parent>

    <groupId>vn.vinhomes</groupId>
    <artifactId>customer-service</artifactId>
    <version>1.0.0-SNAPSHOT</version>

    <properties>
        <java.version>25</java.version>
        <qtvhbds-common.version>1.0.3</qtvhbds-common.version>
    </properties>

    <repositories>
        <repository>
            <id>vhmmarket-vinit-release-reader</id>
            <name>vhmmarket-vinit-release-reader-local</name>
            <url>https://jfrog.vinsmartfuture.tech/artifactory/vsf-qtvhbds-maven-local/vhmmarket</url>
            <layout>default</layout>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>qtvhbds</groupId>
            <artifactId>qtvhbds-common_java25sprb4</artifactId>
            <version>${qtvhbds-common.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

Common được build và kiểm thử với Spring Boot `4.0.2`. Service dùng phiên bản Spring Boot 4.x đã
được team phê duyệt; parent của service tiếp tục quản lý version dependency thực tế trên classpath.
Ví dụ, service dùng parent `4.1.0` sẽ resolve các Spring module transitively từ common về `4.1.0`.

CI/CD phải có repository `vhmmarket-vinit-release-reader` ở trên để Maven tải common từ JFrog.
Credential không ghi trong POM; pipeline cấp qua `settings.xml` với `<server><id>` đúng bằng
`vhmmarket-vinit-release-reader`. Local developer đã cấu hình mirror/repository tương đương trong
Maven settings thì không cần lặp credential trong project.

`groupId`, `artifactId` và `version` ở đầu POM định danh chính service, không thể được truyền từ một
dependency. `artifactId` luôn bắt buộc; `groupId` và `version` chỉ có thể bỏ khi company parent đã
khai báo chúng.

Dependency Maven chỉ truyền thư viện, không truyền `<parent>` hoặc build plugin của common. Vì vậy
service vẫn cần Spring Boot parent/company parent và `spring-boot-maven-plugin` để tạo executable
JAR. Ngoài phần build tối thiểu đó, service chỉ thêm dependency riêng cho nghiệp vụ mà common không
cung cấp, ví dụ MapStruct hoặc SDK của một provider riêng.

### Không khai báo lại các dependency đã có

`vhm-common` là foundation dependency và đã kéo theo các nhóm sau:

| Nhóm | Dependency đã được common cung cấp |
|---|---|
| HTTP | Spring Web MVC, validation, RestClient |
| Persistence | Spring Data JPA, PostgreSQL driver, Liquibase |
| Security/API | Spring Security, Actuator, Springdoc OpenAPI |
| Messaging/cache | Spring Kafka, Spring Cache, Caffeine, Redisson |
| Integration | Apache Thrift 0.24, Commons Pool, Profile và Personal Contact contracts |
| Shared utilities | Jackson, libphonenumber, Apache Commons, Handlebars |

Service không khai báo lại các starter/dependency trên và không pin version khác trong POM nếu
không có quyết định nâng version ở cấp common. Khai báo trùng dễ tạo version drift và làm một service
có hành vi khác baseline của team.

Lombok có scope `provided`; service sử dụng annotation Lombok vẫn khai báo Lombok/annotation
processor trong build của chính service. Tương tự, Hibernate static metamodel hoặc MapStruct
processor thuộc build-time của service và không được truyền qua dependency common.

## Cấu trúc common

```text
vn.vinhomes.common
├── client
│   ├── file
│   │   ├── internal                  secret-key File API
│   │   │   └── dto
│   │   └── privatefile               Basic Auth Private File API
│   │       └── dto
│   ├── market
│   │   └── dto                       Market transport contracts
│   ├── message
│   │   └── dto                       Message Delivery contracts
│   ├── ocr
│   │   └── vinbigdata
│   │       └── dto                   VinBigdata transport contracts
│   ├── contact
│   │   └── personal                  Personal Contact facade
│   ├── profile
│   │   └── generated                 generated Profile Thrift types
│   └── thrift                        pooled Thrift transport infrastructure
├── config                            Spring Boot auto-configurations
├── controller                        BaseController
├── crypto                            AES-GCM compatible cipher
├── dto
│   ├── api                           ApiResponse và ErrorDetails
│   └── page                          PageDto và Pagination
├── entity                            optional JPA mapped superclasses
├── constant                          ErrorCode và ApiCode chuẩn
├── exception                         ApiException và handler
├── repository                        base repository và typed criteria primitives
└── util                              stateless technical utilities
```

Ranh giới package:

- `client` chỉ chứa transport concern. DTO của provider nằm cạnh client tương ứng và không dùng làm
  entity hoặc public API response.
- `config` chỉ lắp ghép bean, bind property và đặt condition; không chứa business behavior.
- `controller`, `dto` và `exception` tạo thành HTTP contract chung nhưng không đặt dưới package
  `web` để tránh thêm một tầng không mang ý nghĩa.
- `repository` là persistence API duy nhất; không tạo thêm package `persistence` song song.
- `entity` chỉ cung cấp base class tùy chọn; không ép mọi entity dùng audit hoặc UUID.
- `util` chỉ nhận logic stateless, tổng quát; provider/domain-specific helper ở lại package sở hữu.
- Không tạo package `core` vì bản thân common đã là tầng dùng chung thấp nhất.

Không đưa vào common:

- Entity, trạng thái hoặc workflow nghiệp vụ.
- Request/response DTO của public API riêng một service.
- Kafka topic, listener hoặc event chỉ một service sở hữu.
- Query/specification chứa business rule riêng.
- Helper chỉ có một consumer.

## Cấu hình common

### Import dependency là đã nhận cấu hình chung

Service **không cần khai báo lại cấu hình chung**. Khi `vhm-common` có trên classpath,
`DefaultsLoader` tự nạp `vhm-common-defaults.yml` với priority thấp; Spring Boot auto-configuration
tiếp tục tạo các bean phù hợp theo classpath và property. Service không cần:

- Copy các block `server`, `management`, virtual thread, multipart, Hikari, JPA, Kafka serializer,
  logging hoặc HTTP timeout từ common vào `application.properties`.
- Thêm `spring.config.import` để import file cấu hình của common.
- Thêm `@Import`, component scan `vn.vinhomes.common` hoặc một annotation bật toàn bộ common.
- Tạo lại các configuration class đã được common cung cấp.

Nếu service chưa dùng capability tùy chọn như web contract, client, Redisson hoặc AES thì **không
cần thêm property nào của common**. Các capability này mặc định không tạo bean. Khi cần, service chỉ
bật đúng capability và truyền giá trị riêng của môi trường; ví dụ:

```properties
# Chỉ khai báo khi service sử dụng Market client
vhm.client.market.enabled=true
```

ConfigMap/Vault cấp `MARKET_CLIENT_BASE_URL`, `MARKET_CLIENT_USERNAME` và
`MARKET_CLIENT_PASSWORD`; common đã map các biến này vào client properties. Ví dụ trên không phải
cấu hình bắt buộc của mọi service. Một service không dùng Market thì bỏ toàn bộ
`vhm.client.market.*`. Mỗi HTTP client được bật độc lập bằng
`vhm.client.<client-name>.enabled=true`; không có switch `vhm.client.enabled` chung.

Tóm lại, file cấu hình của service chỉ chứa ba loại giá trị:

1. Danh tính và tài nguyên riêng của service, như application name, datasource/schema, Kafka
   topic hoặc consumer group.
2. Switch cho capability common mà service thực sự sử dụng.
3. Endpoint và credential theo môi trường, lấy từ ConfigMap/Vault.

### Auto-configuration map

Service không khởi tạo trực tiếp các class trong `vn.vinhomes.common.config`. Spring Boot tự load
chúng từ metadata của JAR:

| Configuration | Điều kiện kích hoạt | Bean/hành vi cung cấp | Service cần khai báo |
|---|---|---|---|
| `CommonConfig` | Có application `ObjectMapper` | `JsonUtil` | Không cần property |
| `CommonConfig` - AES | Có `vhm.crypto.aes-gcm.key` | `AesGcmCipher` | Key và key version |
| `WebConfig` | `web.enabled=true` | Exception handler và optional CORS | Chỉ switch `web.enabled` |
| `WebConfig` - CORS | Web bật và `cors.enabled=true` | Global CORS mapping | Allowed origins của môi trường |
| `SecurityConfig` | `security.basic.enabled=true` | Stateless Basic Auth filter chain | Users từ Vault và path nếu override |
| `AuthContextConfig` | `security.auth-context.enabled=true` | Caller service/source context | Service-to-source mappings |
| `OpenApiConfig` | `springdoc.api-docs.enabled=true` | OpenAPI metadata/security scheme | Title/version theo API |
| `KafkaConfig` | Có Spring Kafka | Manual/batch factories; named clusters | Standard `spring.kafka.*` connection |
| `RedissonConfig` | `redisson.config.enabled=true` | Managed `RedissonClient` | Address và credential môi trường |
| `RestClientConfig` | Từng `vhm.client.<name>.enabled=true` | Typed HTTP client tương ứng | Base URL và credential của client |
| `ThriftClientAutoConfiguration` | `vhm.client.thrift.enabled=true` | Thrift pools, `ProfileClient` và `PersonalContactClient` | Khai báo endpoint cho service thực sự sử dụng |

Các configuration dùng `@ConditionalOnMissingBean` ở extension point phù hợp. Nếu application đã
có bean tùy chỉnh, common không tạo bean cạnh tranh.

### Property kích hoạt capability

| Capability | Property bật | Required khi bật |
|---|---|---|
| Web auto-configuration | `web.enabled=true` | Không; đăng ký exception handler chuẩn |
| CORS | `cors.enabled=true` | Allowed origins production |
| Basic Auth | `security.basic.enabled=true` | `security.basic.users` |
| Auth context | `security.auth-context.enabled=true` | `security.auth-context.sources` nếu cần map source |
| OpenAPI | `springdoc.api-docs.enabled=true` | Không; nên đặt title/version |
| Redisson | `redisson.config.enabled`, `redisson.config.addresses` | Bật capability và có ít nhất một Redis address |
| AES-GCM | `vhm.crypto.aes-gcm.key` | Key và `key-version` |
| File | `vhm.client.file.enabled=true` | Base URL và secret key |
| Private File | `vhm.client.private-file.enabled=true` | Base URL, username, password |
| Market | `vhm.client.market.enabled=true` | Base URL, username, password |
| Message Delivery | `vhm.client.message-delivery.enabled=true` | Base URL, username, password |
| VinBigdata | `vhm.client.vin-bigdata.enabled=true` | Base URL, app ID, app secret |
| Thrift clients | `vhm.client.thrift.enabled=true` | `vhm-profile-mw` và/hoặc `personal-contact` host, port |

Không có `vhm.enabled`, `vhm.common.*` hoặc `vhm.client.enabled`. Mỗi capability có đúng một điểm
kích hoạt để tránh phải bật chồng nhiều switch.

### Default và thứ tự override

Common nạp `vhm-common-defaults.yml` ở priority thấp. Đây là baseline nằm trong JAR, không phải file
service phải copy hoặc import. Thứ tự override là:

1. Command-line argument.
2. Environment variable/Vault/ConfigMap.
3. `application-{profile}.properties` và `application.properties` của service.
4. Default của common.

Không copy toàn bộ default của common vào service. Application chỉ khai báo giá trị đặc thù như
application name, datasource, Liquibase schema/changelog, Kafka connection và upstream credential.

### Những property service không cần khai báo lại

| Nhóm | Baseline common đã cung cấp |
|---|---|
| Server | `server.port=8080` |
| Actuator | Chỉ expose health, bật liveness/readiness probe, health detail theo authorization |
| Threading | Virtual thread bật mặc định |
| Multipart | Tắt mặc định, file `20MB`, request `21MB` |
| Hikari | Pool size, minimum idle, connection/idle/max lifetime |
| JPA/Hibernate | `ddl-auto=validate`, `open-in-view=false`, UTC, JDBC batch, ordered insert/update |
| Liquibase | Tắt mặc định, analytics tắt |
| Kafka | String serializer/deserializer, idempotent producer, no auto commit |
| OpenAPI | API docs và Swagger UI tắt mặc định |
| Logging | Console-only Logback, ECS structured format và correlation `trace_id`/`span_id` |
| HTTP clients | Connect `3s`, read `30s`, mọi client tắt mặc định |

Không lặp các dòng trên trong `application-<env>.properties` nếu service không cần override. Khi
common thay đổi baseline, service sẽ nhận được hành vi mới thay vì bị property copy cũ che mất.

### Những property vẫn thuộc service

| Khi sử dụng | Service phải khai báo |
|---|---|
| Mọi service | `spring.application.name` |
| PostgreSQL/JPA | Datasource URL, username, password và schema |
| Liquibase | `enabled=true`, changelog và schema của service |
| Kafka | ConfigMap cấp `KAFKA_BOOTSTRAP_SERVERS`, client ID/group nếu cần override; service sở hữu topic nghiệp vụ |
| HTTP/Thrift client | Capability `enabled`, endpoint và credential tương ứng |
| Basic Auth | `security.basic.enabled` và users lấy từ Vault |
| Redisson | Addresses; password/TLS/database nếu môi trường yêu cầu |
| AES-GCM | Key và key version |

Nguyên tắc ownership:

- Common sở hữu technical default an toàn và giống nhau giữa các service.
- ConfigMap sở hữu endpoint, topology và giá trị thay đổi theo môi trường.
- Vault/secret manager sở hữu password, token, key và credential.
- Service sở hữu schema, topic, consumer group và các switch capability nó sử dụng.

### Những configuration class service không cần tạo lại

Sau khi import common, không viết lại:

- `RestControllerAdvice` chỉ để map validation/framework exception về response chuẩn.
- `KafkaConfig` chỉ để tạo `ProducerFactory`, `ConsumerFactory`, `KafkaTemplate` hoặc manual listener
  factory cho cluster chính.
- `RestClient`/`HttpServiceProxyFactory` configuration cho các client common đã hỗ trợ.
- `RedissonClient` bean khi dùng đúng `redisson.config.*`.
- OpenAPI bean chỉ để đặt title/version/basic-auth.
- Spring Boot default security exclusion; common đã ngăn generated user/password ngoài ý muốn.

Auto-configuration được Spring Boot phát hiện từ JAR nên service không cần `@Import`, không cần mở
rộng component scan sang `vn.vinhomes.common` và không cần một master `@EnableCommon`. Repository là
ngoại lệ: service dùng custom `BaseRepository` phải thêm `@EnableBaseRepositories` trên main
application để Spring Data chọn `BaseRepositoryImpl`.

Nếu service cần hành vi khác, khai báo bean cùng type/name để common back off; không copy source rồi
sửa thành một phiên bản thứ hai.

Quy tắc namespace:

- Framework giữ namespace chuẩn: `spring.*`, `server.*`, `management.*`, `springdoc.*`.
- Hạ tầng dùng convention kỹ thuật: `security.*`, `cors.*`, `redisson.config.*`, `web.*`.
- Tích hợp Vinhomes dùng `vhm.*`: `vhm.client.*`, `vhm.crypto.*`.
- Không thêm lớp `common`, ví dụ dùng `vhm.client.market`, không dùng `vhm.common.client.market`.

## Khởi tạo HTTP service

`web.enabled=true` đăng ký exception handler chuẩn và optional CORS. Health check dùng endpoint
Actuator; common không tạo health controller riêng.

Controller trả response chuẩn bằng `BaseController`:

```java
@RestController
@RequestMapping("/v1/customers")
final class CustomerController extends BaseController {
    private final CustomerService service;

    CustomerController(CustomerService service) {
        this.service = service;
    }

    @GetMapping
    ApiResponse<PageDto<CustomerResponse>> search(CustomerSearch request, Pageable pageable) {
        return success(service.search(request, pageable));
    }
}
```

Repository/service trả `Page<T>`; chỉ chuyển sang `PageDto<T>` tại API boundary.

## Error code của service

`ApiCode` dành cho lỗi hạ tầng chung. Mỗi service định nghĩa `constant.AppErrorCode` bằng cách
implement `ErrorCode`:

- Dải `10000-10999` được common giữ cho lỗi framework/hạ tầng chuẩn.
- Mã từ `20000` trở lên thuộc service và phải theo dải được team phân bổ; không tái sử dụng mã của
  `ApiCode` cho lỗi nghiệp vụ.

```java
public enum AppErrorCode implements ErrorCode {
    NOT_FOUND(20404, 404, "Customer {0} không tồn tại");

    private final int code;
    private final int httpStatusCode;
    private final String message;

    AppErrorCode(int code, int httpStatusCode, String message) {
        this.code = code;
        this.httpStatusCode = httpStatusCode;
        this.message = message;
    }

    @Override public int code() { return code; }
    @Override public int httpStatusCode() { return httpStatusCode; }
    @Override public String message() { return message; }
}
```

```java
throw new ApiException(AppErrorCode.NOT_FOUND, customerId);
```

Placeholder message dùng `{0}`, `{1}`. Không trả exception message nội bộ, SQL hoặc upstream
credential cho API consumer.

## Repository và entity

### Các thành phần và trách nhiệm

| Thành phần | Nhiệm vụ | Dev sử dụng khi |
|---|---|---|
| `EnableBaseRepositories` | Bật Spring Data repository với implementation chung | Đặt một lần trên main application |
| `BaseRepository<T, ID>` | Contract repository chung, kế thừa `JpaRepository` và `JpaSpecificationExecutor` | Khai báo repository của entity và gọi page, limit, projection, lock, bulk/native API |
| `BaseRepositoryImpl<T, ID>` | Implementation nội bộ của `BaseRepository` | Không inject, không extends và không khởi tạo trực tiếp |
| `JpaCriteria<T>` | Builder immutable để viết chuỗi điều kiện type-safe trên nhiều field | Query đơn giản dạng `status = ... AND createdAt >= ...` |
| `JpaAttribute<T, V>` | Đại diện thao tác trên đúng một JPA metamodel attribute | Tạo predicate, sort, assignment hoặc attribute projection dùng lại |
| `JpaSpecs` | Thêm/bỏ optional specification an toàn theo input | Request có filter nullable, collection rỗng hoặc keyword blank |
| `JpaFilters` | Chuyển record có `@JpaFilter` thành specification | Filter CRUD phẳng có nhiều field optional |
| `JpaAssignment<T>` | Mô tả một cặp field/value cho Criteria bulk update | Gọi `updateMatching` hoặc set field không implement `Comparable` |
| `SqlQuery` | Gói native SQL và named parameters bất biến | Cần PostgreSQL-specific SQL, join/CTE/grouping khó biểu diễn bằng Criteria |

Quy tắc chọn nhanh:

- Bắt đầu query đơn giản bằng `repository.where(...)` — kết quả là `JpaCriteria`.
- Cần thao tác lặp lại trên một attribute thì dùng `repository.attribute(...)` — kết quả là `JpaAttribute`.
- Input optional thì bọc predicate bằng `JpaSpecs`.
- Join, subquery hoặc business rule phức tạp thì viết `Specification` có tên trong service.
- Chỉ xuống `SqlQuery` khi JPA Specification không còn phù hợp.

`JpaCriteria` và `JpaAttribute` không thay thế nhau. `JpaCriteria` giữ toàn bộ biểu thức đã chain;
`JpaAttribute` chỉ giữ metadata của một attribute và có thể sinh ra một predicate, `Sort.Order` hoặc
`JpaAssignment`. Ví dụ:

```java
JpaCriteria<Customer> activeRecently = customerRepository
        .where(Customer_.status).eq(Status.ACTIVE)
        .and(Customer_.createdAt).ge(createdFrom);

JpaAttribute<Customer, Instant> createdAt = customerRepository.attribute(Customer_.createdAt);
Sort newestFirst = Sort.by(createdAt.desc());
```

### 1. Kích hoạt base repository

Không cần tạo `JpaConfig` riêng. Đặt annotation trực tiếp trên main application; mặc định Spring
scan repository từ package của application trở xuống:

```java
@SpringBootApplication
@EnableBaseRepositories
public class CustomerApplication {
}
```

`@EnableBaseRepositories` thay cho `@EnableJpaRepositories` và đăng ký
`BaseRepositoryImpl` làm implementation chung. Không khai báo đồng thời hai annotation cho cùng một
package.

Chỉ thêm `@EnableJpaAuditing` khi entity dùng `AuditedEntity`, `AuditedUuidEntity`, `@CreatedDate`
hoặc `@LastModifiedDate`:

```java
@SpringBootApplication
@EnableBaseRepositories
@EnableJpaAuditing
public class CustomerApplication {
}
```

Service multi-module hoặc có repository nằm ngoài root package mới cần giới hạn package rõ ràng:

```java
@EnableBaseRepositories(basePackageClasses = CustomerRepository.class)
```

```java
interface CustomerRepository extends BaseRepository<Customer, UUID> {
}
```

Repository của service vẫn có thể khai báo derived query hoặc `@Query` riêng. `BaseRepository` chỉ
cung cấp primitive kỹ thuật dùng lại; không chứa query nghiệp vụ của service.

### 2. Static metamodel và criteria cơ bản

`Customer_`, `Order_`... phải được Hibernate annotation processor sinh lúc compile, không viết tay.
`vhm-common` không thể truyền annotation processor sang build của application, vì vậy mỗi service
dùng repository API phải thêm `hibernate-processor` vào `maven-compiler-plugin`, giống cấu hình sau:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </path>
            <path>
                <groupId>org.hibernate.orm</groupId>
                <artifactId>hibernate-processor</artifactId>
                <version>${hibernate.version}</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

Sau `mvn compile`, các metamodel nằm trong `target/generated-sources/annotations`, ví dụ
`Customer_`. Không tạo class `_` bằng tay và không commit thư mục `target`. IDE cần bật annotation
processing hoặc import lại Maven project để nhận generated source. Metamodel giúp lỗi sai tên/type
attribute xuất hiện lúc compile thay vì lúc chạy:

```java
var condition = customerRepository.where(Customer_.status).eq(Status.ACTIVE)
        .and(Customer_.createdAt).ge(createdFrom);

Page<Customer> page = customerRepository.findAll(condition, pageable);
```

Các operator có sẵn: `eq`, `notEq`, `in`, `notIn`, `lt`, `le`, `ge`, `gt`, `isNull` và
`isNotNull`. Chain `and`/`or` kết hợp với toàn bộ biểu thức đứng trước; nếu cần grouping phức tạp,
hãy tách thành các `Specification` có tên rõ nghĩa rồi ghép bằng `and`, `or`,
`Specification.allOf(...)` hoặc `Specification.anyOf(...)`.

`Customer_.status` chỉ là metadata của field, chưa gắn với SQL root nên không thể gọi trực tiếp
`Customer_.status.in(...)`. Dùng `repository.where(Customer_.status)` để tạo chain hoặc
`repository.attribute(Customer_.status)` để lấy predicate/sort/assignment cho attribute đó.

### 3. Filter động và pagination

Không dùng cờ boolean kiểu `statusFilter`, `regionFilter` để tạo một JPQL method có hàng chục
parameter. Mỗi filter optional trở thành một specification độc lập:

```java
var status = customerRepository.attribute(Customer_.status);
var regionId = customerRepository.attribute(Customer_.regionId);

Specification<Customer> condition = Specification.allOf(
        JpaSpecs.whenPresent(request.status(), status::eq),
        JpaSpecs.whenNotEmpty(request.regionIds(), regionId::in),
        JpaSpecs.whenHasText(request.keyword(), CustomerSpecs::matchesKeyword));

Page<Customer> page = customerRepository.findAll(condition, pageable);
```

| Helper | Khi nào thêm predicate |
|---|---|
| `JpaSpecs.when(enabled, supplier)` | Cờ điều kiện là `true` |
| `JpaSpecs.whenPresent(value, factory)` | Giá trị khác `null` |
| `JpaSpecs.whenNotEmpty(values, factory)` | Collection còn ít nhất một phần tử khác `null` |
| `JpaSpecs.whenHasText(value, factory)` | Chuỗi sau trim không rỗng |

Helper bị skip trả `Specification.unrestricted()`, vì vậy có thể compose trực tiếp và không cần
kiểm tra `null`.

Với filter CRUD phẳng, có thể khai báo mapping ngay trên record thay vì lặp lại các phép kiểm tra
null/blank và dựng predicate thủ công:

```java
@JpaFilterFor(Customer.class)
public record CustomerFilter(
        @JpaFilter(path = "status", operator = IN) List<Status> statuses,
        @JpaFilter(path = "createdAt", operator = GE) Instant createdFrom,
        @JpaFilter(path = "createdAt", operator = LE) Instant createdTo,
        @JpaFilter(path = "regionId") String regionId) {
}
```

```java
Page<Customer> page = customerRepository.findAll(JpaFilters.from(filter), pageable);
```

`JpaFilters` chỉ đọc record component có annotation. Giá trị `null`, string blank, collection rỗng
hoặc chỉ chứa `null` được bỏ qua; `CONTAINS` và `STARTS_WITH` escape ký tự SQL LIKE và so sánh
không phân biệt hoa thường. Metadata được cache sau lần dùng đầu tiên, đồng thời path/operator sai
thất bại trước khi query được thực thi. Các operator hỗ trợ: `EQ`, `NOT_EQ`, `IN`, `NOT_IN`, `GT`,
`GE`, `LT`, `LE`, `CONTAINS`, `STARTS_WITH`, `IS_NULL`, `IS_NOT_NULL`.

Không dùng annotation cho authorization, join collection, subquery, JSONB hoặc nhóm logic phức tạp.
Các điều kiện đó vẫn là specification có tên rõ nghĩa và được compose với filter:

```java
var condition = JpaFilters.<Customer>from(filter)
        .and(CustomerSpecs.visibleTo(currentUser))
        .and(CustomerSpecs.currentAttempt());
```

Repository luôn trả Spring `Page<T>`. Chỉ chuyển sang response chuẩn tại controller boundary:

```java
@GetMapping
ApiResponse<PageDto<CustomerResponse>> search(CustomerSearch request, Pageable pageable) {
    Page<CustomerResponse> result = service.search(request, pageable);
    return success(result);
}
```

Không để `PageDto` chảy ngược xuống repository/service persistence.

### 4. Complex specification thuộc service

Join, subquery, JSONB và quy tắc chọn bản ghi ưu tiên là business query nên đặt trong class như
`CustomerSpecs`, không đưa vào common:

```java
final class CustomerSpecs {
    static Specification<Customer> belongsToAgency(UUID agencyId) {
        return (customer, query, criteria) -> {
            var agency = customer.join(Customer_.agencies);
            query.distinct(true);
            return criteria.equal(agency.get(Agency_.id), agencyId);
        };
    }

    static Specification<Customer> matchesKeyword(String keyword) {
        String pattern = "%" + SqlUtil.escapeLike(keyword.toLowerCase(Locale.ROOT)) + "%";
        return (customer, query, criteria) -> criteria.like(
                criteria.lower(customer.get(Customer_.fullName)), pattern, '\\');
    }

    private CustomerSpecs() {
    }
}
```

Đặt tên specification theo ý nghĩa nghiệp vụ (`currentAttempt`, `belongsToAgency`,
`matchesKeyword`), không đặt theo cách cài đặt như `query1` hoặc `jsonFilter`.

### 5. Read helper và projection

| API | Mục đích |
|---|---|
| `findFirst(condition, sort)` | Lấy record đầu tiên theo sort xác định |
| `findAll(condition, sort, limit)` | Lấy tối đa `limit` entity, không cần total count |
| `findIds(condition, sort, limit)` | Chỉ select ID, không hydrate entity |
| `findFirstId(condition)` | Lấy một ID phù hợp |
| `findMax(field, condition)` | Lấy giá trị lớn nhất của một field |
| `findDistinct(field, condition)` | Lấy các giá trị field không trùng |

Khi kết quả “đầu tiên” có ý nghĩa, luôn truyền sort ổn định, nên có thêm ID làm tie-breaker:

```java
var order = Sort.by(
        customerRepository.attribute(Customer_.createdAt).desc(),
        customerRepository.attribute(Customer_.id).desc());

Optional<Customer> latest = customerRepository.findFirst(condition, order);
```

### 6. Pessimistic locking và worker cạnh tranh

`findByIdForUpdate` và `findAllForUpdate` chờ lock đang bị giữ. `findAllForUpdateSkipLocked` bỏ qua
record đã bị worker khác lock, phù hợp để claim batch công việc:

```java
@Transactional
public List<Job> claim(int batchSize) {
    var pending = jobRepository.where(Job_.status).eq(JobStatus.PENDING);
    var oldestFirst = Sort.by(
            jobRepository.attribute(Job_.createdAt).asc(),
            jobRepository.attribute(Job_.id).asc());

    var jobs = jobRepository.findAllForUpdateSkipLocked(pending, oldestFirst, batchSize);
    jobs.forEach(job -> job.markProcessing(workerId));
    return jobs;
}
```

Các method lock phải chạy trong transaction nghiệp vụ. Lock được nhả khi transaction kết thúc;
không đọc locked entity rồi xử lý claim ở ngoài transaction mà chưa cập nhật trạng thái sở hữu.

### 7. Bulk update và delete

Bulk update dùng `PredicateSpecification<T>` để condition không phụ thuộc `CriteriaQuery`:

```java
int affected = customerRepository.updateMatching(
        (customer, criteria) -> criteria.equal(customer.get(Customer_.status), Status.PENDING),
        List.of(
                customerRepository.attribute(Customer_.status).set(Status.ACTIVE),
                JpaAssignment.set(Customer_.updatedAt, Instant.now())));
```

`updateMatching` flush trước và clear persistence context sau khi update. Nó bỏ qua entity callback,
JPA auditing và optimistic locking tự động; service phải tự set audit field và thêm version predicate
nếu contract yêu cầu.

`deleteMatching(condition, sort, limit)` claim một batch bằng `SKIP LOCKED` rồi xóa bulk. Method này
không chạy `@PreRemove` hoặc cascade theo từng managed entity; dùng `deleteAll(...)` nếu cần callback.

### 8. Native SQL và pagination

Chỉ dùng native SQL khi Criteria/JPQL không biểu diễn tốt PostgreSQL feature cần thiết. SQL thuộc
service và mọi request value phải bind bằng named parameter:

```java
var query = new SqlQuery("""
        SELECT c.*
        FROM customer c
        JOIN agency_customer ac ON ac.customer_id = c.id
        WHERE ac.agency_id = :agencyId
          AND c.status = :status
        ORDER BY c.created_at DESC, c.id DESC
        """, Map.of("agencyId", agencyId, "status", Status.ACTIVE.name()));

Page<Customer> page = customerRepository.findPage(query, pageable);
```

`findPage(query, pageable)` tự sinh count bằng cách wrap toàn bộ SQL thành derived table. Với query
group/CTE nặng hoặc không thể wrap an toàn, truyền count riêng:

```java
Page<Customer> page = customerRepository.findPage(dataQuery, countQuery, pageable);
```

Native query phải select đủ column để Hibernate map về root entity. `Pageable` chỉ cung cấp offset
và page size; sort phải viết rõ trong SQL. Không nối keyword, sort column hoặc filter value từ
request vào statement.

### 9. Chọn entity base

Chọn entity base theo schema:

| Nhu cầu | Base class |
|---|---|
| UUIDv7 | `BaseEntityUUID` |
| Audit, ID tự định nghĩa | `AuditedEntity` |
| UUIDv7 và audit | `AuditedUuidEntity` |

Entity không phù hợp ba contract trên thì không cần extends base class.

## Security, CORS và OpenAPI

Basic Auth là capability độc lập:

```properties
security.basic.enabled=true
```

ConfigMap/Vault truyền trực tiếp `SECURITY_BASIC_USERS`, `SECURITY_BASIC_PATH_PATTERNS`,
`SECURITY_BASIC_PUBLIC_PATH_PATTERNS` và `SECURITY_BASIC_DOCUMENTATION_PATH_PATTERNS`; service
không cần tạo lại các dòng mapping `${SECURITY_*}`. `SECURITY_BASIC_USERS` có dạng
`username:password[:role|role]`, nhiều tài khoản phân tách bằng dấu phẩy và phải đến từ
Vault/secret. Service có `SecurityFilterChain` riêng thì common tự back off; không bật Basic Auth
chỉ để dùng web response hoặc health.

Caller context là capability riêng:

```properties
security.auth-context.enabled=true
```

ConfigMap truyền `SECURITY_AUTH_CONTEXT_SOURCES=service-a=source-a,service-b=source-b`. Không dùng
lại tên cũ `SECURITY_BASIC_SOURCES` vì source mapping không thuộc Basic Auth.

```properties
cors.enabled=${CORS_ENABLED:true}
cors.allowed-origin-patterns=${CORS_ALLOWED_ORIGIN_PATTERNS:https://*.vinhomes.vn}
springdoc.api-docs.enabled=${SPRINGDOC_API_DOCS_ENABLED:true}
springdoc.swagger-ui.enabled=${SPRINGDOC_SWAGGER_UI_ENABLED:true}
web.openapi.title=Customer API
web.openapi.version=v1
web.openapi.basic-auth=false
web.openapi.server-url=${WEB_OPENAPI_SERVER_URL:}
```

## Observability và Kubernetes probes

Runtime sử dụng OpenTelemetry Java agent đã được đóng gói trong image của team. Deployment kích
hoạt agent qua `JAVA_TOOL_OPTS`; common không đóng gói thêm tracing SDK, exporter hoặc HTTP/Kafka
instrumentation để tránh tạo span hai lần.

`ObservationRegistry` trong common chỉ cung cấp instrumentation hook; nó không tự gửi dữ liệu đến
Grafana. Mỗi deployment cần cấu hình Java agent và OTLP exporter trong ConfigMap, ví dụ môi trường
staging:

```yaml
JAVA_TOOL_OPTS: "-javaagent:/usr/app/opentelemetry-javaagent.jar"
OTEL_EXPORTER_OTLP_ENDPOINT: "http://k8s-monitoring-traces-collector.monitoring.svc.cluster.local:4317"
OTEL_SERVICE_NAME: "customer-service"
OTEL_RESOURCE_ATTRIBUTES: "deployment.environment=stag"
OTEL_SEMCONV_STABILITY_OPT_IN: "database"
OTEL_EXPORTER_OTLP_PROTOCOL: "grpc"
OTEL_TRACES_SAMPLER: "parentbased_traceidratio"
OTEL_TRACES_SAMPLER_ARG: "1.0"
OTEL_TRACES_EXPORTER: "otlp"
OTEL_METRICS_EXPORTER: "otlp"
OTEL_LOGS_EXPORTER: "none"
OTEL_INSTRUMENTATION_MICROMETER_ENABLED: "true"
```

`OTEL_SERVICE_NAME` và `deployment.environment` phải theo service/môi trường triển khai, không copy
nguyên giá trị ví dụ. `OTEL_EXPORTER_OTLP_HEADERS` là secret và phải lấy từ Vault, không commit vào
manifest hoặc repository. Sampling `1.0` phù hợp staging/debug; production dùng tỷ lệ do team
observability quy định.

Agent chịu trách nhiệm truyền trace context qua servlet, `RestClient` và Kafka, đồng thời đưa
`trace_id`/`span_id` vào MDC. Common dùng đúng các key này trong ECS correlation pattern và đặt
`traceId` vào metadata của error response. Không đưa provider message, URL query hoặc credential
vào response.

Kubernetes dùng Actuator probe chuẩn:

```text
/actuator/health/liveness
/actuator/health/readiness
```

Docker healthcheck dùng `/actuator/health`. Kubernetes dùng `/actuator/health/liveness` cho
liveness probe và `/actuator/health/readiness` cho readiness probe. Database, Redis và Kafka health
được quản lý bởi Actuator health contributor tương ứng; service không tự viết controller để ping
từng dependency.

## Kafka

Common kéo `spring-boot-starter-kafka`, không chỉ thư viện `spring-kafka`. Vì vậy Spring Boot tạo
primary `KafkaTemplate`, `ProducerFactory` và `ConsumerFactory`; `KafkaConfig` của common chỉ bổ
sung manual/batch listener factory và các named cluster. Service không khai báo lại Kafka starter
hoặc cấu hình bean cho primary cluster.

Cluster chính dùng namespace chuẩn của Spring Boot:

```properties
spring.kafka.bootstrap-servers=${KAFKA_BOOTSTRAP_SERVERS}
spring.kafka.client-id=${KAFKA_CLIENT_ID:${spring.application.name}}
spring.kafka.consumer.group-id=${KAFKA_CONSUMER_GROUP:${spring.application.name}}
```

Listener manual chọn `manualCommitKafkaListenerContainerFactory`; batch listener chọn
`batchManualCommitKafkaListenerContainerFactory`.

Cluster bổ sung khai báo theo logical name:

```properties
spring.kafka.clusters.history.bootstrap-servers=${HISTORY_KAFKA_BOOTSTRAP_SERVERS}
spring.kafka.clusters.history.client-id=${HISTORY_KAFKA_CLIENT_ID:customer-history}
spring.kafka.clusters.history.consumer.group-id=${HISTORY_KAFKA_CONSUMER_GROUP}
spring.kafka.clusters.history.listener.concurrency=${HISTORY_KAFKA_CONCURRENCY:2}
spring.kafka.clusters.history.listener.ack-mode=MANUAL_IMMEDIATE
```

Entry `history` tạo các bean có prefix `history`, ví dụ `historyKafkaTemplate` và
`historyKafkaListenerContainerFactory`. Inject named bean bằng `@Qualifier`.

Common không tự đặt consumer group `default`. Cluster có consumer phải có group ID riêng; cluster
chỉ dùng producer không cần group ID.

## Outbound clients

Mỗi client có một switch riêng và mặc định tắt:

```properties
vhm.client.connect-timeout=${VHM_CLIENT_CONNECT_TIMEOUT:3s}
vhm.client.read-timeout=${VHM_CLIENT_READ_TIMEOUT:30s}
vhm.client.market.enabled=true
```

Các switch hiện có:

- `vhm.client.file.enabled`
- `vhm.client.private-file.enabled`
- `vhm.client.market.enabled`
- `vhm.client.message-delivery.enabled`
- `vhm.client.vin-bigdata.enabled`
- `vhm.client.thrift.enabled`

Base URL/host/port lấy từ ConfigMap; secret/password/app credential lấy từ Vault. Khi client bật,
required property được validate lúc startup.

Private File dùng các biến môi trường thống nhất sau; service không khai báo lại mapping trong
`application.yml`/`application.properties`:

```properties
FILE_PRIVATE_CLIENT_BASE_URL=...
FILE_PRIVATE_CLIENT_USERNAME=...
FILE_PRIVATE_CLIENT_PASSWORD=...
FILE_PRIVATE_CLIENT_CATEGORY=default
```

Các HTTP client (`MarketClient`, `FileClient`, `PrivateFileClient`, `MessageDeliveryClient` và
`VinBigdataClient`) đều là Spring HTTP Interface. Service chỉ inject interface cần dùng;
`RestClientConfig` tự build transport và tạo proxy. Không tạo lại `RestClient`, interceptor hoặc
`HttpServiceProxyFactory` trong service:

```java
@Service
@RequiredArgsConstructor
class ProjectLookupService {
    private final MarketClient marketClient;

    MarketProjectDetailResponse findProject(String projectId) {
        return marketClient.getProjectById(projectId);
    }
}
```

Method của client nhận/trả transport DTO. Mapping sang domain, validation nghiệp vụ, token lifecycle,
retry, circuit breaker, fallback và cache thuộc service sử dụng. Riêng `FileClient`, secret key được
`RestClientConfig` gắn tự động; caller chỉ truyền request DTO. `PrivateFileClient` nhận thêm
`type` vì đây là path variable của File Service.

Mọi HTTP client dùng `RestClient.Builder` của Spring Boot, gắn `ObservationRegistry`, timeout chung
và safe metadata logging. Log chỉ chứa client, method, endpoint không có query, status và duration;
không log header, request/response body hoặc credential. Để observation được export lên Grafana,
deployment vẫn phải cấu hình OpenTelemetry agent và OTLP exporter như mục Observability ở trên.

HTTP error được chuẩn hóa thành `ClientException`; HTTP 408, 429 và 5xx có `retryable=true`.
Connection/read timeout giữ exception transport chuẩn của Spring và được `ApiExceptionHandler` map
thành lỗi dependency unavailable. Cờ `retryable` chỉ là phân loại lỗi, common không tự retry request.

## Redisson

```properties
REDISSON_ENABLED=true
REDISSON_CONFIG_ADDRESSES=redis.internal:6379
REDISSON_CONFIG_PASSWORD=
REDISSON_CONFIG_DATABASE=0
REDISSON_CONFIG_KEY_PREFIX=customer-service:
```

Common chỉ tạo `RedissonClient` khi `REDISSON_ENABLED=true`. Cluster mode bật bằng
`REDISSON_CONFIG_USE_CLUSTER_SERVERS=true` và khai báo nhiều address.

## AES-GCM

```properties
vhm.crypto.aes-gcm.key=${AES_GCM_KEY}
vhm.crypto.aes-gcm.key-version=${AES_GCM_KEY_VERSION}
```

`key` là Base64 raw AES key 128/192/256 bit. Ciphertext format là
`12-byte nonce || ciphertext || 16-byte authentication tag`.

Không thay `AesGcmCipher` bằng Spring `Encryptors.stronger(...)` nếu đã có dữ liệu mã hóa: Spring
dùng key derivation và ciphertext format khác nên không decrypt được dữ liệu hiện hữu. Application
phải lưu `key-version` cạnh ciphertext nếu hỗ trợ key rotation.

## Quy tắc mở rộng common

Chỉ bổ sung code khi đáp ứng cả ba điều kiện:

1. Có từ hai service thực sự sử dụng hoặc đã được thống nhất thành team contract.
2. Không chứa business semantics của một domain.
3. Có API rõ ràng, test và default an toàn; không tạo runtime resource khi chưa bật.

Application có thể thay bean mặc định bằng bean cùng type/name. Ưu tiên cơ chế back-off này thay vì
exclude toàn bộ auto-configuration.

## Build

```bash
mvn clean verify
```

Trước khi release cần compile ít nhất một service Web/JPA và một service Kafka/client với artifact
mới. Không commit hoặc phát hành thay đổi phá vỡ contract `qtvhbds.common.*` trong dòng `1.x`.
