# NOXH Java Platform

> Nền tảng dùng chung cho các service Java 25 / Spring Boot 4 của Vinhomes.
>
> Tài liệu này mô tả cấu trúc mới, vai trò của từng library, ranh giới ownership và cách
> quyết định một thành phần nên nằm trong library hay service.

## 1. Vì sao cần platform dùng chung?

### 1.1. Hiện trạng của cấu trúc cũ

Trong cấu trúc cũ, mỗi service đồng thời sở hữu hai hệ thống code:

- **Nghiệp vụ:** entity, workflow, rule, repository query, API, scheduler và event của domain.
- **Nền tảng:** Maven dependency, base entity, UUID generator, Kafka, Redis, exception handler,
  API response, security, REST client, logging và utility.

Hai nhóm này lại được đặt chung trong namespace của service:

```text
vn/vinhomes/<service>/
├── controller/ dto/ model/ repository/ service/  # nghiệp vụ
├── annotation/ config/ exception/ security/       # nền tảng bị copy
├── kafka/ redis/ util/                             # nền tảng bị copy
└── client/                                         # contract upstream bị khóa trong service
```

Nhìn vào package không thể xác định đâu là code riêng của domain, đâu là implementation chung vô
tình được tạo đầu tiên trong repository đó. Khi service khác cần cùng capability, lựa chọn nhanh nhất
là copy code. Mỗi bản copy sau đó có lifecycle riêng và trở thành một biến thể mới.

### 1.2. Nền tảng bị nhân bản và sai lệch hành vi

Các class như base entity, UUID generator, API response, global exception handler, Kafka config,
Redis config và security filter xuất hiện ở nhiều service. Trùng tên không có nghĩa trùng hành vi:

- cùng một exception có thể map sang HTTP status hoặc error payload khác nhau;
- cùng một DTO có thể khác field name, validation hoặc Jackson serialization;
- cùng một Kafka config có thể khác serializer, retry, acknowledgement và consumer factory;
- cùng một Redis config có thể khác prefix, timeout hoặc cách bật cluster;
- cùng một security filter có thể khác path matcher hoặc cách kiểm tra signature.

Sai lệch này thường không gây compile error. Nó chỉ xuất hiện khi tích hợp hoặc vận hành, nên chi phí
phát hiện cao hơn duplication thông thường.

Ví dụ, một lỗi được sửa trong `RestControllerExceptionHandler` của dossier không tự động được sửa
trong OCR hoặc campaign. Sau vài lần thay đổi, không còn câu trả lời rõ ràng cho câu hỏi “bản nào là
chuẩn?”.

### 1.3. Outbound client bị gắn sai ownership

File, OCR, Market, IAM, Message Delivery và Profile là upstream contract, nhưng client và DTO cũ
thường nằm dưới package của service đầu tiên cần chúng. Điều này tạo ra ba vấn đề:

1. Service khác khó phát hiện client đã tồn tại nên tiếp tục tạo hoặc copy client mới.
2. Upstream thay đổi contract phải rà soát nhiều repository và dễ bỏ sót consumer.
3. DTO của upstream nằm cạnh domain model khiến developer sử dụng trực tiếp DTO transport làm
   business model, làm domain bị phụ thuộc vào contract bên ngoài.

Generated Thrift code còn làm duplication lớn hơn: mỗi service dùng Profile có thể phải giữ hàng
chục nghìn dòng generated source và tự duy trì transport pool/configuration giống nhau.

### 1.4. Security bị phân mảnh

Mỗi service tự có `SecurityConfig`, filter, basic-auth setup, CIDR rule và actor-context handling.
Hệ quả không chỉ là code lặp mà còn là rủi ro bảo mật:

- service có thể bảo vệ thiếu path mới;
- cùng một internal request được xác thực khác nhau giữa các service;
- logic chống replay hoặc kiểm tra clock skew được cập nhật không đồng đều;
- cấu hình mở rộng phải sửa Java code và fork cả filter chain;
- package crypto/signing dễ bị hiểu nhầm với web authentication/authorization.

Security là capability cần một implementation chuẩn, còn khác biệt về realm, path và credential phải
được điều khiển bằng cấu hình.

### 1.5. Kafka và Redis có nhiều nguồn cấu hình

Cấu trúc cũ tồn tại đồng thời các namespace và implementation khác nhau, ví dụ
`spring.kafka.*`, `kafka.*`, Kafka config riêng của service và legacy config. Một property có thể được
khai báo nhưng không được bean đang chạy bind tới.

Hệ quả:

- khó xác định serializer/factory nào đang thực sự được sử dụng;
- local, test và production có thể khởi tạo khác nhau;
- listener không dùng vẫn có thể tạo connection khi startup;
- sửa timeout, retry hoặc security protocol phải làm nhiều nơi;
- service mới tiếp tục copy một cấu hình bất kỳ mà không biết đó có phải cấu hình chuẩn hay không.

Redis gặp vấn đề tương tự với standalone/cluster, TLS, key prefix và connection pool. Mục tiêu mới là
một implementation dùng chung cho mỗi capability, còn service chỉ cung cấp giá trị YAML.

### 1.6. Maven dependency và version phân tán

Mỗi service cũ tự khai báo Spring Boot starters, library version, annotation processors, test
dependency và build plugin. Các POM dài nhưng vẫn không thể hiện rõ phần nào là platform baseline,
phần nào thực sự phục vụ nghiệp vụ.

Tác động trực tiếp:

- nâng Spring Boot hoặc Java phải sửa từng repository;
- vá CVE không biết còn service nào chưa nâng version;
- Lombok/MapStruct/Hibernate processor có thể chạy khác nhau;
- test runner và packaging behavior không nhất quán;
- dependency conflict được xử lý cục bộ và lặp lại ở nhiều service;
- build chạy được trên một máy nhưng thất bại trong CI do phụ thuộc relative path hoặc artifact chỉ
  được install local.

### 1.7. Cấu hình bị copy và không rõ độ ưu tiên

Một service có thể mang cả default kỹ thuật, giá trị local và cấu hình môi trường trong cùng file.
Khi copy sang service khác, port, schema, topic, key prefix hoặc credential placeholder cũng bị copy
theo.

Không có quy tắc ownership khiến developer khó trả lời:

- property này do library hay service định nghĩa;
- giá trị mặc định nằm ở đâu;
- property nào bắt buộc khi feature được bật;
- secret lấy từ Vault hay đang có default không an toàn;
- vì sao test lại đọc cấu hình local của developer.

Kết quả là lỗi cấu hình chỉ được phát hiện lúc startup hoặc sau khi kết nối nhầm resource.

### 1.8. Test và local environment trở nên nặng, khó tái lập

Khi service tự mang PostgreSQL, Redis, Kafka, ZooKeeper và Docker Maven configuration giống nhau,
mỗi repository phải duy trì lifecycle test infrastructure riêng. Plugin có thể tự khởi động container
cho cả test không cần integration environment, làm build chậm và phụ thuộc Docker trên máy chạy.

Các default thiếu điều kiện còn khiến application fail startup chỉ vì một capability không được dùng
nhưng vẫn cố tạo datasource, Kafka consumer, Redis connection hoặc Thrift pool.

### 1.9. Onboarding và review phụ thuộc trí nhớ cá nhân

Developer mới phải hỏi hoặc tìm một repository để copy trước khi viết nghiệp vụ:

- response envelope chuẩn là class nào;
- exception nào map sang status nào;
- entity nên kế thừa base class nào;
- File/Profile client đã có chưa;
- Kafka listener dùng factory nào;
- security mở path hoặc thêm realm bằng cách nào.

Reviewer cũng phải đọc sâu implementation mới biết một thay đổi là domain-specific hay đang tạo thêm
một bản platform mới. Kiến thức nằm trong trí nhớ của người làm lâu năm thay vì nằm trong module
boundary và API có tài liệu.

### 1.10. Chi phí tăng theo số service và số biến thể

Vấn đề lớn nhất không phải số dòng code bị copy, mà là số implementation phải duy trì. Nếu có `N`
service và mỗi capability có nhiều biến thể, một thay đổi nền tảng tạo ra chuỗi công việc:

```text
phân tích tất cả biến thể
        → sửa từng repository
        → test từng implementation
        → release theo nhiều lịch khác nhau
        → theo dõi service chưa được cập nhật
```

Một bản vá có thể đúng ở ba service nhưng bị bỏ sót ở service thứ tư. Chi phí và rủi ro tăng cùng số
repository, trong khi giá trị nghiệp vụ không tăng tương ứng.

### 1.11. Nguyên nhân gốc

> Cấu trúc cũ không có ranh giới ownership rõ ràng giữa platform, inbound web, outbound integration
> và business domain.

Do đó, giải pháp không thể chỉ là xóa class trùng hoặc tạo thêm một package `util`. Cần tách theo
trách nhiệm và dependency direction:

| Điểm yếu cũ | Yêu cầu kiến trúc mới | Thành phần chịu trách nhiệm |
|---|---|---|
| Version/plugin khác nhau | Một build baseline có version | `vhm-spring-boot-parent` |
| Primitive, JPA, Kafka, Redis bị copy | Một implementation hạ tầng thấp nhất | `vhm-common` |
| Response, exception và security phân mảnh | Một inbound HTTP contract có thể cấu hình | `vhm-web-starter` |
| Client/DTO nằm rải rác trong service | Typed client được nhóm theo upstream | `vhm-client` |
| Business và infrastructure trộn lẫn | Service chỉ giữ hành vi của domain | Từng service |

Cấu trúc mới vì vậy tách rõ:

- **Platform libraries** sở hữu capability kỹ thuật dùng chung, có version và lifecycle riêng.
- **Service** sở hữu toàn bộ hành vi sản phẩm và nghiệp vụ của domain mình.

Mục tiêu không phải đưa càng nhiều code vào `common` càng tốt. Mục tiêu là giảm số biến thể phải
bảo trì, tạo một implementation chính thức cho mỗi capability, nhưng không biến platform thành một
monolith dùng chung mới.

### 1.12. Áp dụng cấu trúc mới thì giảm được gì?

Thay đổi quan trọng nhất là chuyển từ **nhiều bản sao không có owner** sang **một contract có owner
và version**. Giá trị không chỉ nằm ở số dòng code được xóa mà ở số nơi phải sửa, kiểm thử và theo
dõi khi hệ thống thay đổi.

| Công việc | Cấu trúc cũ | Cấu trúc mới | Lợi ích trực tiếp |
|---|---|---|---|
| Vá lỗi Kafka/Redis/base infrastructure | Tìm và sửa từng service | Sửa một library, phát hành version, các service nâng version | Giảm implementation phải sửa và nguy cơ bỏ sót |
| Thay đổi API response/exception mapping | Mỗi service có thể trả contract khác nhau | Một contract trong `vhm-web-starter` | Client nhận response nhất quán hơn |
| Cập nhật security mechanism | Sửa nhiều filter chain | Sửa implementation chung, khác biệt khai báo bằng YAML | Giảm security drift và phạm vi audit |
| Upstream thay đổi contract | Tìm client/DTO trong nhiều repository | Sửa typed client theo capability | Biết rõ consumer và nơi chịu trách nhiệm |
| Nâng Java/Spring/dependency | Chỉnh nhiều POM và plugin | Quản lý baseline tại parent | Nâng cấp, vá CVE và rollback có kiểm soát |
| Tạo service mới | Copy một service gần giống rồi xóa bớt | Kế thừa baseline và chỉ viết domain | Ít code khởi tạo, ít lỗi copy nhầm |
| Review business feature | Phải lọc qua nhiều infrastructure class | Service chủ yếu còn code domain | Review nhanh hơn, blast radius rõ hơn |
| Onboarding developer | Hỏi người cũ hoặc tìm code để copy | Tìm API theo module/package và đọc JavaDoc | Giảm phụ thuộc kiến thức truyền miệng |

#### Giảm duplication có thể đo được

Kết quả refactor trên dossier và OCR/eKYC cho thấy tác động thực tế:

| Chỉ số | Mức giảm |
|---|---:|
| Dossier Java production files | 89 file |
| Dossier Java LOC | 18% |
| Dossier POM | 88% số dòng |
| OCR/eKYC Java production files | 40 file |
| OCR/eKYC Java LOC | 26% |
| OCR/eKYC POM | 85% số dòng |
| Generated Profile Thrift trong từng service | Không còn nhân bản; chuyển về `vhm-client` |

Số liệu trên không có nghĩa code nền tảng biến mất. Code dùng chung được chuyển tới một artifact và
code trùng bị loại bỏ. Phần giảm quan trọng nhất là **số bản implementation phải duy trì**.

#### Giảm chi phí thay đổi

Ví dụ khi cần sửa cách verify internal request:

```text
Cấu trúc cũ
  sửa dossier → test/release dossier
  sửa OCR     → test/release OCR
  sửa campaign→ test/release campaign
  rà soát xem còn bản copy nào khác

Cấu trúc mới
  sửa + test vhm-web-starter một lần
  publish version mới
  từng service nâng version theo kế hoạch và chạy compatibility test
```

Cấu trúc mới không loại bỏ việc test và release service. Nó loại bỏ việc viết lại cùng một bản sửa
ở từng repository, đồng thời cho phép theo dõi adoption bằng version dependency.

#### Giảm rủi ro vận hành

- Kafka và Redis có một nơi cấu hình serializer, timeout, connection và default behavior.
- Security có một filter chain chuẩn; path, realm và credential thay đổi bằng cấu hình.
- Capability tắt không được tạo connection/pool không cần thiết khi startup.
- Property có namespace và owner, giảm nguy cơ khai báo một key nhưng runtime đọc key khác.
- Rollback library bằng cách pin version cũ rõ ràng hơn rollback nhiều bản sửa thủ công.
- Contract upstream tập trung giúp đánh giá phạm vi ảnh hưởng trước khi triển khai.

#### Làm rõ phạm vi thay đổi và ownership

Với cấu trúc mới, vị trí code thể hiện loại thay đổi:

- thay đổi trong service thường chỉ ảnh hưởng một domain;
- thay đổi trong `vhm-client` ảnh hưởng consumer của upstream tương ứng;
- thay đổi trong `vhm-web-starter` ảnh hưởng inbound HTTP contract;
- thay đổi trong `vhm-common` có phạm vi rộng nhất và phải được kiểm thử nghiêm ngặt;
- thay đổi trong parent ảnh hưởng build/dependency baseline.

Điều này giúp reviewer, QA và release owner xác định blast radius mà không cần đọc toàn bộ source.

#### Lợi ích theo từng vai trò

| Vai trò | Lợi ích |
|---|---|
| Developer | Viết ít boilerplate, không phải đoán implementation chuẩn, API dùng chung có JavaDoc |
| Reviewer | Dễ phát hiện business leakage và thay đổi sai module, phạm vi review rõ hơn |
| QA | Contract nhất quán, test chung cho infrastructure và test service tập trung vào use case |
| DevOps/SRE | Property namespace thống nhất, version triển khai truy vết được, ít biến thể runtime |
| Security | Một nơi audit/cập nhật cơ chế dùng chung, giảm cấu hình lệch giữa service |
| Tech lead/Manager | Đo adoption bằng version, ước lượng phạm vi nâng cấp và ownership rõ ràng |

#### Lợi ích không đến miễn phí

Shared library làm giảm duplication nhưng tăng mức ảnh hưởng của một thay đổi sai. Vì vậy cấu trúc
mới chỉ hiệu quả khi:

- library giữ backward compatibility hoặc công bố breaking change rõ ràng;
- có test tại library và compatibility test tại consumer;
- service chủ động nâng version, không dùng version mutable trong production;
- không đưa business rule vào common chỉ vì muốn tái sử dụng;
- capability lớn được tách tiếp khi classpath hoặc ownership trở nên quá rộng.

Đổi lại, tổ chức quản lý một số artifact có kỷ luật thay vì quản lý nhiều bản sao không kiểm soát.

## 2. Cấu trúc tổng thể

```text
new-structure/
├── libraries/
│   ├── vhm-spring-boot-parent/  # Maven/build baseline
│   ├── vhm-common/              # primitive và infrastructure dùng chung
│   ├── vhm-web-starter/         # inbound HTTP contract và web security
│   └── vhm-client/              # outbound clients và upstream contracts
└── service/
    ├── vhm-dossier-core/        # domain hồ sơ
    ├── vhm-ocr-ekyc/            # domain OCR/eKYC
    └── vhm-campaign-core/       # domain campaign
```

Quan hệ phụ thuộc runtime:

```text
                         service
                            │
              ┌─────────────┴─────────────┐
              v                           v
      vhm-web-starter                vhm-client
              │                           │
              └─────────────┬─────────────┘
                            v
                       vhm-common
```

Các quy tắc bắt buộc:

1. Library không phụ thuộc ngược vào service.
2. `vhm-common` không phụ thuộc `vhm-web-starter` hoặc `vhm-client`.
3. `vhm-client` không phụ thuộc `vhm-web-starter`.
4. Business entity, business error code, trạng thái và quyết định nghiệp vụ luôn nằm ở service.
5. Capability có kết nối ra ngoài phải có điều kiện bật/tắt; capability không dùng không được làm
   service fail startup.
6. Khác biệt giữa các service ưu tiên biểu diễn bằng YAML/property, không copy config class.

## 3. Vai trò và trách nhiệm từng library

### 3.1. `vhm-spring-boot-parent`

#### Vai trò

Đây là Maven parent thống nhất cách build và dependency baseline của các service. Module có
`packaging=pom`, không chứa Java source và không cung cấp business API.

Parent hiện chịu trách nhiệm:

- thống nhất Java 25 và Spring Boot 4;
- quản lý version override và bản vá dependency tập trung;
- cấu hình Lombok, MapStruct và Hibernate annotation processor;
- cấu hình Maven Compiler, Surefire và Spring Boot Maven Plugin;
- cung cấp test dependency chuẩn như Spring Boot Test, Kafka Test, Testcontainers và H2;
- khai báo Maven repository dùng chung;
- cung cấp version đồng bộ cho các VHM library.

#### Cách dùng trong service độc lập

```xml
<parent>
    <groupId>vn.vinhomes.platform</groupId>
    <artifactId>vhm-spring-boot-parent</artifactId>
    <version>0.1.0-SNAPSHOT</version>
    <relativePath/>
</parent>
```

Service nằm ở Git repository riêng phải resolve parent đã publish từ Maven repository.
`<relativePath/>` ngăn Maven tìm một đường dẫn local như
`../../libraries/vhm-spring-boot-parent/pom.xml`, vốn không tồn tại trong CI/CD.

#### Không thuộc trách nhiệm của parent

- Java class và application configuration;
- entity, DTO, business dependency hoặc plugin chỉ phục vụ một domain;
- secret và giá trị cấu hình theo môi trường;
- logic giúp POM ngắn hơn nhưng làm mọi service nhận dependency không cần thiết.

#### Trạng thái hiện tại

Parent hiện là **opinionated application parent**: nó khai báo trực tiếp `vhm-common`,
`vhm-web-starter`, `vhm-client` và test dependencies. Cách này phù hợp với baseline HTTP service và
giúp POM service ngắn, nhưng worker không dùng web vẫn nhận web classpath.

Khi có nhiều loại application, nên tách thành một BOM/parent chỉ quản version/plugin và để service
chọn starter cần dùng. Không nên mô tả parent hiện tại là “chỉ quản build” cho tới khi hoàn thành
việc tách đó.

---

### 3.2. `vhm-common`

#### Vai trò

`vhm-common` là lớp nền thấp nhất. Module sở hữu primitive và infrastructure có thể dùng lại,
không gắn với inbound HTTP, một upstream cụ thể hoặc business domain cụ thể.

#### Trong library có gì?

| Package | Thành phần | Trách nhiệm |
|---|---|---|
| `vn.vinhomes.common.annotation` | `UUIDv7Generated`, UUID/String generators | Chuẩn sinh định danh UUIDv7 cho entity |
| `vn.vinhomes.common.entity` | `BaseEntity`, `BaseEntityUUID`, `AuditedEntity*` | Field persistence và audit nền |
| `vn.vinhomes.common.repository` | base repository, condition/assignment builders, `@EnablePersistence` | Primitive JPA dùng chung, không chứa query nghiệp vụ |
| `vn.vinhomes.common.config` | cache, Kafka, Kafka/Jackson, legacy Kafka, Redisson | Auto-configure hạ tầng từ property |
| `vn.vinhomes.common.crypto` | canonical request, HMAC signer/verifier, cipher, signature headers | Primitive ký và mã hóa độc lập với web security |
| `vn.vinhomes.common.util` | file, hash, HTTP, JSON, validation, phone, masking, string, UUID | Utility thuần không quyết định business flow |

Tên `crypto` được dùng có chủ ý. Đây là công cụ ký/mã hóa; gọi package này là `security` sẽ dễ bị
nhầm với authentication/authorization do `vhm-web-starter` sở hữu.

#### Auto-configuration và default

`CommonAutoConfiguration` là entry point của module và import các config dùng chung. Default nằm
trong `vhm-common-defaults.yml`, bao gồm:

- virtual threads;
- datasource, HikariCP, JPA và Liquibase baseline;
- Kafka producer/consumer và legacy Kafka compatibility;
- Redis/Redisson;
- structured logging.

`LibraryDefaultsEnvironmentPostProcessor` nạp default library ở độ ưu tiên thấp để service và
environment luôn có thể override. Các cipher chỉ được tạo khi có key tương ứng trong
`vhm.crypto.*`.

#### Dependency nền được cung cấp

Module hiện cung cấp baseline cho validation, actuator, JPA/Liquibase/PostgreSQL, cache/Caffeine,
Kafka, Redisson, Apache HttpClient 5 và Apache POI. Việc dependency nằm ở common chỉ
chuẩn hóa implementation/version; nó không phải lý do để đưa mọi code sử dụng dependency đó vào
common.

#### Không được đặt trong `vhm-common`

- controller, servlet filter, API response hoặc HTTP status mapping;
- client và DTO của một upstream cụ thể;
- business error code, domain enum, domain entity và query nghiệp vụ;
- workflow, scheduler hoặc Kafka listener xử lý nghiệp vụ;
- secret hay URL của một môi trường cụ thể.

---

### 3.3. `vhm-web-starter`

#### Vai trò

`vhm-web-starter` chuẩn hóa **inbound HTTP boundary** cho Spring MVC service. Nó sở hữu contract web
chung, exception-to-response mapping, timezone request context, OpenAPI và một security
configuration thống nhất.

#### Trong library có gì?

| Package | Thành phần | Trách nhiệm |
|---|---|---|
| `vn.vinhomes.web.controller` | `BaseController`, `HttpResponse`, health endpoint | Primitive/controller chung cho HTTP boundary |
| `vn.vinhomes.web.dto` | `ApiResponse`, metadata, paging, `ServiceResponse` | Envelope và pagination contract chuẩn |
| `vn.vinhomes.web.exception` | generic exceptions, error payload, global handler | Chuyển lỗi thành HTTP response nhất quán |
| `vn.vinhomes.web.config` | `SecurityConfig`, timezone-aware Jackson | Wiring web dùng chung |
| `vn.vinhomes.web.security.realm` | realm properties, CIDR filter | Basic-auth realm và network allowlist |
| `vn.vinhomes.web.security.internal` | HMAC filter, replay guard, actor context | Xác thực request nội bộ và chống replay |
| `vn.vinhomes.web.security.upstream` | BFF registry/filter, current actor, data scope | Nhận identity từ BFF và chống giả mạo header chéo IdP |
| `vn.vinhomes.web.timezone` | filter và request timezone context | Chuẩn hóa timezone theo request |
| `vn.vinhomes.web.util` | `IpUtil` | Xử lý IP phục vụ web/security |

#### Một `SecurityConfig`, khác biệt đi qua YAML

Service không tạo thêm một `SecurityConfig` cạnh config của starter. Realm, path, CIDR, internal
signature, actor context và upstream BFF được khai báo bằng property:

```yaml
security:
  basic: ...
  realms: ...
  internal-signature: ...
  internal-actor-context: ...

vhm:
  web:
    openapi: ...
    security:
      upstream-bff: ...
  cors: ...
```

Chỉ override bean khi service có một security mechanism thực sự khác contract chung. Feature mở
rộng phải có `enabled` hoặc property rõ ràng, không fork/copy toàn bộ config.

`WebAutoConfiguration` cung cấp OpenAPI metadata/group, CORS, global exception handler, timezone
configuration và các web bean mặc định. Bean có `@ConditionalOnMissingBean` có thể được thay thế có
kiểm soát; feature có `@ConditionalOnProperty` chỉ bật khi YAML yêu cầu.

#### Ranh giới exception

Starter sở hữu **cơ chế** và generic HTTP exceptions như bad request, forbidden, invalid state và
resource not found. Service sở hữu **vocabulary lỗi nghiệp vụ** và điều kiện ném lỗi. Ví dụ
`CampaignErrorCode` hoặc `DossierApprovalException` phải nằm ở service, dù global handler của starter
chịu trách nhiệm serialize response.

#### Không được đặt trong `vhm-web-starter`

- outbound REST/Thrift client;
- JPA entity hoặc repository;
- quyết định phân quyền gắn với workflow domain;
- request/response DTO chỉ thuộc một API nghiệp vụ;
- credential thật.

#### Trạng thái hiện tại

`HealthController` vẫn đang được `WebAutoConfiguration` đăng ký. Nếu contract cuối cùng chỉ dùng
Spring Boot Actuator thì cần xóa class và bean này bằng một thay đổi code riêng; không được giả định
nó đã bị bỏ.

---

### 3.4. `vhm-client`

#### Vai trò

`vhm-client` là outbound integration layer dùng chung. Module đóng gói cách kết nối,
authentication, timeout, transport và DTO theo contract của upstream. Service inject typed client,
quyết định khi nào gọi và map kết quả vào domain của mình.

#### Cấu trúc theo capability

Client, properties, exception và DTO của cùng upstream được đặt cạnh nhau:

```text
vn.vinhomes.client/
├── file/                   # File Service public/private + DTO
├── iam/                    # IAM Workforce Core
├── incom/                  # Incom API + DTO
├── market/                 # Market API + properties + DTO
├── message/                # Message Delivery + properties + DTO
├── ocr/                    # OCR contract
│   ├── dto/
│   ├── provider/dto/
│   └── vinbigdata/         # VinBigData properties + DTO
├── profile/                # typed facade gọi Profile
└── thrift/                 # factory, validator, pool và transport config
```

Package gốc `vn.vinhomes.client` chỉ chứa infrastructure dùng chung:

- `RestClientSupport` và `RestClients`;
- basic authentication;
- `ClientException`;
- `ClientAutoConfiguration`.

Không tạo lại `client.config` hoặc `client.dto` chứa lẫn thành phần của nhiều upstream. Developer
phải có thể mở một package capability và thấy client, config, exception cùng DTO liên quan.

Generated Thrift contract nằm dưới `vn.vinhomes.service.agent_profile`. Đây là generated source,
không phải business code và không chỉnh tay.

#### Cách sử dụng client

1. Cấu hình đúng property group; credential do Vault/environment cung cấp.
2. Inject typed client.
3. Gọi operation của upstream.
4. Map response/exception sang model và outcome của domain trong service.

Ví dụ File client:

```yaml
vhm:
  client:
    file:
      base-url: ${FILE_CLIENT_BASE_URL}
      secret-key: ${FILE_CLIENT_SECRET_KEY}
      username: ${FILE_PRIVATE_CLIENT_USERNAME}
      password: ${FILE_PRIVATE_CLIENT_PASSWORD}
      connect-timeout: 3s
      read-timeout: 20s
```

```java
@Service
final class AttachmentService {
    private final FileClient fileClient;

    AttachmentService(FileClient fileClient) {
        this.fileClient = fileClient;
    }
}
```

URL có thể có default local an toàn; username, password, HMAC key và API key thật không commit vào
repository. Vault hoặc secret manager inject environment variable được YAML tham chiếu.

#### Khi nào một client được đưa vào library?

Một client nên vào `vhm-client` khi:

- contract upstream ổn định;
- có ít nhất hai consumer thực tế, hoặc là platform capability được nhận ownership;
- API không lộ model của service đầu tiên;
- timeout, authentication và error semantics có thể chuẩn hóa.

Client chỉ phục vụ một domain thì ở lại service. Một class có hậu tố `Client` không tự động là
shared code.

#### Không được đặt trong `vhm-client`

- quyết định khi nào gửi notification, duyệt dossier hoặc chạy campaign;
- fallback làm thay đổi business outcome;
- domain entity, controller DTO hoặc inbound HTTP exception;
- web security filter chain.

## 4. Service repository chịu trách nhiệm gì?

Sau khi dùng platform, service vẫn sở hữu toàn bộ hành vi sản phẩm:

```text
src/main/java/vn/vinhomes/<domain>/
├── controller/       # endpoint riêng của domain
├── dto/              # request/response của domain API
├── model|entity/     # domain state và persistence model
├── repository/       # query mang business semantics
├── service/          # use case và business rules
├── mapper/           # mapping domain/application
├── event|kafka/      # event schema/listener/publisher của domain
├── scheduler/        # job nghiệp vụ
├── client/           # adapter chỉ service này dùng
├── config/           # extension thật sự đặc thù
└── exception/        # business error code/exception
```

Service cũng sở hữu:

- Liquibase changelog cho schema của mình;
- topic name, consumer group, concurrency và payload nghiệp vụ;
- endpoint mapping theo môi trường;
- permission vocabulary, feature flag và rate limit của domain;
- test cho use case, migration và integration boundary;
- Dockerfile và deployment manifest riêng.

Platform cung cấp Kafka/Redis machinery; service sở hữu topic và cách xử lý message. Platform cung
cấp security mechanism; service sở hữu permission và policy nghiệp vụ. Platform cung cấp client;
service sở hữu quyết định gọi client và xử lý kết quả.

## 5. Bảng quyết định đặt code

Áp dụng theo thứ tự và dừng ở điều kiện đầu tiên khớp:

| Câu hỏi | Vị trí |
|---|---|
| Có thuật ngữ, trạng thái hoặc quyết định của một domain? | Service sở hữu domain |
| Mô tả contract/transport của upstream dùng chung? | `vhm-client/<capability>` |
| Xử lý inbound HTTP giống nhau giữa các service? | `vhm-web-starter` |
| Là primitive kỹ thuật, không phụ thuộc HTTP và không có business rule? | `vhm-common` |
| Chỉ quản version, plugin hoặc build baseline? | `vhm-spring-boot-parent` |
| Chưa có consumer thứ hai và không phải platform capability? | Giữ tại service |

Ví dụ cụ thể:

| Thành phần | Vị trí đúng | Lý do |
|---|---|---|
| `UUIDv7Generator` | `vhm-common` | Primitive persistence |
| `CampaignStatus` | campaign service | Trạng thái nghiệp vụ |
| `RestControllerExceptionHandler` | `vhm-web-starter` | HTTP mapping dùng chung |
| `CampaignErrorCode` | campaign service | Vocabulary lỗi domain |
| `FileClient` và File DTO | `vhm-client/file` | Upstream contract dùng chung |
| Chọn loại tài liệu dossier cần upload | dossier service | Quyết định nghiệp vụ |
| Kafka producer factory | `vhm-common` | Infrastructure chung |
| Notification topic/listener | campaign service | Event contract và workflow domain |
| HMAC signer | `vhm-common.crypto` | Crypto primitive |
| Servlet HMAC authentication filter | `vhm-web-starter.security` | Inbound web security |

Một class trùng tên ở hai service chưa đủ điều kiện để share. Chỉ gom khi semantics, lifecycle và
owner thực sự giống nhau.

## 6. Cấu hình và ownership

Cấu hình được chia thành ba lớp ưu tiên:

```text
Vault / Kubernetes Secret / environment variable    ưu tiên cao nhất
                    ↓
application.yml hoặc application-<profile>.yml      service/environment
                    ↓
vhm-*-defaults.yml trong library                    default thấp nhất
```

Nguyên tắc:

- Library default phải an toàn và không chứa secret.
- Service chỉ override phần khác biệt, không copy toàn bộ default library.
- Secret dùng `${ENV_NAME}` và được Vault/deployment inject.
- Feature tùy chọn phải có `enabled` hoặc điều kiện bean rõ ràng.
- Local profile nằm ở service nếu nó mô tả port/database/schema local của service.
- Không sửa changeset Liquibase đã chạy; tạo changeset mới. Việc tắt Liquibase local không sửa được
  checksum ở môi trường dùng chung.

Namespace ownership:

| Namespace | Owner |
|---|---|
| `spring.datasource`, `spring.jpa`, `spring.liquibase` | common cung cấp baseline; service cung cấp URL/schema |
| `kafka.*`, `redisson.config.*` | `vhm-common` |
| `vhm.crypto.*` | `vhm-common.crypto` |
| `security.*`, `vhm.web.*`, `vhm.cors.*`, `springdoc.*` | `vhm-web-starter` |
| `vhm.client.<capability>.*`, Thrift properties | `vhm-client` |
| `campaign.*`, `dossier.*`, `segment.*`, ... | service tương ứng |

## 7. Auto-configuration và mở rộng

Các runtime library đăng ký entry point qua Spring Boot
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

- `CommonAutoConfiguration`;
- `WebAutoConfiguration`;
- `ClientAutoConfiguration` và Thrift auto-configuration.

Khi thêm bean vào library:

1. Dùng `@ConditionalOnClass` nếu capability cần dependency tùy chọn.
2. Dùng `@ConditionalOnProperty` nếu capability có thể bật/tắt.
3. Dùng `@ConditionalOnMissingBean` khi service được phép thay implementation.
4. Validate property bắt buộc khi feature bật; lỗi phải nêu đúng tên property thiếu.
5. Không mở connection, pool hoặc thread khi capability tắt.
6. Test cả trạng thái bật và tắt.

Service nên ưu tiên cấu hình YAML trước khi override bean. Override là escape hatch cho behavior thực
sự khác, không phải cách cấu hình thông thường.

## 8. JavaDoc cho public library API

Public class trong library phải có JavaDoc đủ để developer sử dụng mà không cần đọc implementation.
JavaDoc cần trả lời:

- class giải quyết và không giải quyết việc gì;
- property bắt buộc/tùy chọn và nguồn secret;
- cách inject hoặc khởi tạo;
- ví dụ gọi ngắn;
- điều kiện auto-configuration;
- lỗi quan trọng, thread-safety hoặc lifecycle nếu có.

Ví dụ JavaDoc cho File client:

```java
/**
 * Xin presigned URL từ File Service; không upload/download byte thay caller.
 *
 * <p>Cấu hình {@code vhm.client.file.base-url}; inject secret bằng
 * {@code FILE_CLIENT_SECRET_KEY}. Private API cần thêm
 * {@code FILE_PRIVATE_CLIENT_USERNAME/PASSWORD} từ Vault.
 *
 * <pre>{@code
 * @Service
 * final class AttachmentService {
 *     AttachmentService(FileClient fileClient) { ... }
 *     // fileClient.prepareUpload(...)
 * }
 * }</pre>
 */
```

Tài liệu kiến trúc như file này giải thích boundary. JavaDoc cạnh class giải thích cách dùng chính
class đó. Không cần duy trì một `GUIDE.md` lặp lại API của từng class.

## 9. Publish và sử dụng giữa các Git repository

Library và service nằm ở các Git repository độc lập, vì vậy quy trình đúng là:

1. Build và test platform libraries.
2. Publish parent POM cùng các JAR lên Maven repository nội bộ.
3. Service pin một platform version đã publish và dùng `<relativePath/>` rỗng.
4. CI service nhận credential đọc Maven registry qua `settings.xml` hoặc CI variable.
5. Build service trong môi trường sạch.
6. Nâng library version qua merge request có changelog và compatibility test.

Local development có thể chạy `mvn install` để đưa snapshot vào `~/.m2`, nhưng đây không phải cơ
chế CI/CD. Build chỉ chạy trên máy đã install local mà chưa publish artifact là build không tái lập.

Với release, không dùng version mutable. Với snapshot, Maven repository cần snapshot policy rõ ràng.

## 10. Quy trình thêm shared capability

Trước khi move code từ service vào library:

1. Xác định consumer thực tế và owner bảo trì.
2. Chứng minh API không lộ business model của service đầu tiên.
3. Chọn đúng library bằng bảng ở mục 5.
4. Thiết kế namespace property và default an toàn.
5. Thêm auto-configuration có điều kiện nếu cần.
6. Viết unit/integration test và JavaDoc hướng dẫn dùng.
7. Kiểm tra dependency direction, compatibility và classpath impact.
8. Publish version mới và migrate từng consumer.
9. Chỉ xóa implementation cũ sau khi service build/test thành công.

## 11. Kết quả refactor đã đo

| Chỉ số | Trước | Sau | Thay đổi |
|---|---:|---:|---:|
| Dossier Java files (`src/main`) | 310 | 221 | −89 |
| Dossier Java LOC | 26.463 | 21.611 | −18% |
| Dossier POM | 436 dòng | 51 dòng | −88% |
| OCR/eKYC Java files | 106 | 66 | −40 |
| OCR/eKYC Java LOC | 5.695 | 4.206 | −26% |
| OCR/eKYC POM | 124 dòng | 19 dòng | −85% |
| Thrift generated code trong mỗi service dùng Profile | khoảng 66.500 dòng | 0 | chuyển về `vhm-client` |

LOC giảm là kết quả của việc bỏ duplication, không phải mục tiêu độc lập. `vhm-client` vẫn có LOC
lớn do generated Thrift contract; không dùng con số đó để đánh giá độ phức tạp code viết tay.

Đã kiểm chứng trong workspace:

- libraries reactor build/install thành công;
- service resolve parent từ Maven repository thử nghiệm mà không cần relative path;
- `vhm-client` không phụ thuộc `vhm-web-starter`;
- namespace shared code thống nhất dưới `vn.vinhomes.*`;
- OCR/eKYC load local profile và datasource khi có cấu hình.

## 12. Technical debt phát hiện khi review

Tài liệu mô tả code hiện tại và ghi nhận rõ các điểm chưa đạt kiến trúc đích:

1. **Parent đang kéo mọi runtime library.** Phù hợp với HTTP service hiện tại nhưng chưa tối ưu cho
   worker; cân nhắc BOM + opt-in starter khi có consumer thực tế.
2. **`HealthController` vẫn còn trong `vhm-web-starter`.** Nếu Actuator là health contract duy nhất,
   cần xóa class và bean đăng ký.
3. **`logback-spring.xml` vẫn còn trong `vhm-common`.** Nếu logging chỉ cấu hình bằng
   YAML/deployment, cần bỏ resource và test lại precedence.
4. **`LegacyKafkaConfig` vẫn tồn tại.** Cần deadline migrate consumer rồi xóa legacy path để đạt mục
   tiêu một Kafka configuration chuẩn.
5. **Classpath common khá rộng** do JPA, Liquibase, Kafka, Redis, POI và HTTP client. Nếu service nhẹ
   bị ảnh hưởng, tách starter theo capability thay vì tiếp tục đưa mọi dependency vào common.

Technical debt phải được quản lý công khai để platform không quay lại trạng thái “mọi thứ đều là
common”.

## 13. Definition of Done cho service đã migrate

Một service chỉ được xem là migrate hoàn tất khi:

- parent dùng version đã publish và `<relativePath/>` rỗng;
- không copy base entity, UUID generator, Kafka/Redis config, web exception handler hoặc shared
  security config;
- client/DTO upstream dùng chung đến từ `vhm-client` và được nhóm theo capability;
- service chỉ giữ business exception, model, repository, workflow và adapter đặc thù;
- cấu hình mở rộng đi qua YAML/property, secret đến từ Vault/environment;
- package shared dùng `vn.vinhomes.*`;
- public library API có JavaDoc hướng dẫn cấu hình và sử dụng;
- `mvn verify` chạy thành công trong môi trường sạch.

Kết quả mong muốn là developer chuyển từ **“tìm một repository để copy”** sang **“dùng một contract
có owner, version và hướng dẫn sử dụng”**.
