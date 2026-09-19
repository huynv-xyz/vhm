# Báo cáo trước và sau khi áp dụng VHM Common Library

**Service:** `vhm-ocr-ekyc`  
**Common artifact:** `qtvhbds:qtvhbds-common_java25sprb4:1.0.3`  
**Ngày đánh giá:** 19/09/2026

## 1. Phạm vi so sánh

- **Trước:** `HEAD` của repository `vhm-ocr-ekyc` trước migration.
- **Sau:** working tree sau khi tích hợp common library.
- Không tính `.env.local` vì đây là cấu hình máy cá nhân và đã được Git ignore.

## 2. Số liệu tổng quan

| Chỉ số | Trước | Sau | Thay đổi |
|---|---:|---:|---:|
| Production Java files | 112 | 69 | Giảm 43 file (38,4%) |
| Production Java LOC | 6.099 | 4.449 | Giảm 1.650 dòng (27,1%) |
| Direct Maven dependencies | 17 | 8 | Giảm 9 dependency (52,9%) |
| `application.yml` | 144 dòng | 69 dòng | Giảm 75 dòng (52,1%) |
| Tổng diff | — | 316 dòng thêm, 2.366 dòng xóa | Giảm ròng 2.050 dòng |

Riêng production code, config, POM và Dockerfile:

- 80 file bị tác động.
- 248 dòng thêm.
- 2.156 dòng xóa.
- Giảm ròng 1.908 dòng.

## 3. Kiến trúc trước khi áp dụng common

OCR tự sở hữu và bảo trì các thành phần hạ tầng sau:

- UUIDv7 annotation và generator.
- `BaseEntity`, `BaseEntityUUID`.
- Base repository và Criteria builder.
- API response DTO.
- Exception và global exception handler.
- Basic Auth security.
- `AuthContextProvider`.
- File Management client và DTO.
- HTTP logging interceptor.
- AES-GCM cipher.
- Utility HTTP, JSON, string, hash, file và UUID.
- Health controller.
- OpenAPI configuration.
- JPA configuration.
- Logback encoder và masking layout.
- Spring, Kafka, datasource, logging và management defaults.

Hệ quả:

- Infrastructure bị sao chép giữa các service.
- Error response, pagination, security và repository behavior dễ lệch chuẩn.
- Fix framework phải thực hiện lặp lại ở nhiều repository.
- POM và cấu hình ứng dụng dài, khó phân biệt phần hạ tầng với phần nghiệp vụ.

## 4. Kiến trúc sau khi áp dụng common

### 4.1 API response và exception

OCR sử dụng trực tiếp từ common:

- `ApiResponse`
- `PageDto`
- `ErrorCode`
- `ApiException`
- `ApiExceptionHandler`
- `BaseController`

Đã xóa implementation riêng:

- `dto/common/ApiResponse`
- `dto/common/ApiMeta`
- `exception/ApiException`
- `RestControllerExceptionHandler`
- Các exception wrapper không còn tạo thêm giá trị.

### 4.2 Persistence

Repository OCR hiện sử dụng:

- `BaseRepository`
- `BaseRepositoryImpl`
- `JpaCriteria`
- `JpaAttribute`
- `JpaAssignment`
- `EnableBaseRepositories`

Đã xóa toàn bộ base repository riêng của OCR:

- `JpaAssignment`
- `JpaColumn`
- `JpaConditionBuilder`
- `JpaRepositoryBase`
- `JpaRepositoryBaseImpl`

Hibernate processor tự sinh static metamodel. Repository scan hiện nhận đủ 6 repository.

### 4.3 Entity

Entity sử dụng `vn.vinhomes.common.entity.BaseEntityUUID`.

Đã xóa:

- `UUIDv7Generated`
- `UUIDv7Generator`
- `BaseEntity`
- `BaseEntityUUID`

Timestamp sử dụng trực tiếp Hibernate:

- `@CreationTimestamp`
- `@UpdateTimestamp`

Service không còn cần `JpaConfig` hoặc `@EnableJpaAuditing`.

### 4.4 HTTP client

Common cung cấp transport policy thống nhất:

- Connect/read timeout.
- `ObservationRegistry`.
- Safe HTTP metadata logging.
- Request factory.
- HTTP-interface proxy creation.
- API-key header.
- Endpoint sanitization.

Đã xóa khỏi OCR:

- `HttpLoggingInterceptor` riêng.
- `FileManagementClient` riêng.
- 9 File Management DTO riêng.

### 4.5 Security

Đã chuyển sang common:

- HTTP Basic authentication.
- Security error response chuẩn.
- `AuthContextProvider`.
- Service-to-source mapping.

Đã xóa package security riêng của OCR.

Runtime config sử dụng ConfigMap/Vault hoặc `.env.local`:

```properties
SECURITY_BASIC_ENABLED=true
SECURITY_BASIC_USERS=...
SECURITY_BASIC_PATH_PATTERNS=/v1/**
SECURITY_BASIC_PUBLIC_PATH_PATTERNS=...
SECURITY_AUTH_CONTEXT_ENABLED=true
SECURITY_AUTH_CONTEXT_SOURCES=...
```

### 4.6 Health và logging

Đã xóa `HealthController`. Health endpoint chuẩn:

```text
GET /actuator/health
```

Docker healthcheck:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl --fail --silent --show-error \
  "http://localhost:${SERVER_PORT:-8080}/actuator/health" || exit 1
```

Logging chung do common quản lý:

- Console logging phù hợp container.
- ECS structured format.
- Trace/span correlation.
- Không tạo rolling log file trong container.

### 4.7 Kafka

Common sử dụng `spring-boot-starter-kafka`, cho phép Spring Boot 4 tự tạo:

- `ProducerFactory`
- `ConsumerFactory`
- `KafkaTemplate`
- Listener container factory

Service không cần tự viết `KafkaConfig`.

### 4.8 Maven

Dependency production trung tâm của OCR:

```xml
<dependency>
    <groupId>qtvhbds</groupId>
    <artifactId>qtvhbds-common_java25sprb4</artifactId>
    <version>1.0.3</version>
</dependency>
```

Springdoc, PostgreSQL, Kafka, Security, JPA, RestClient và Actuator được common cung cấp.

Các dependency còn lại chủ yếu phục vụ build hoặc test:

- Lombok.
- Hibernate static metamodel processor.
- Spring test modules.
- Testcontainers.
- H2.

## 5. Kết quả kiểm chứng runtime

Đã chạy service bằng profile `local`.

Kết quả:

- Application khởi động thành công trên port `8083`.
- Kết nối PostgreSQL thành công.
- Hibernate validate schema local thành công.
- Scan đủ 6 JPA repository.
- Tạo `KafkaTemplate` thành công.
- Kafka consumer kết nối và nhận partition thành công.
- Basic Auth khởi tạo thành công.
- Health endpoint trả HTTP `200`.

Health response:

```json
{
  "groups": ["liveness", "readiness"],
  "status": "UP"
}
```

Common library:

- 60 test pass.
- Build và local install thành công.

## 6. Trạng thái test OCR

Full test suite hiện chưa xanh hoàn toàn:

```text
Tests run: 143
Failures: 2
Errors: 27
```

Các error chủ yếu là lỗi dây chuyền từ hai nguyên nhân gốc:

1. Web slice test nạp repository configuration từ main application nhưng không tạo `EntityManagerFactory`.
2. `JpaPersistenceTest` bật schema validation trước khi test database có bảng `ocr_ekyc_media_refs`.

Hai assertion failure còn lại do test vẫn kỳ vọng message tiếng Anh cũ:

```text
Unexpected error
Request input is invalid
```

Trong khi common trả message chuẩn hiện tại:

```text
Rất tiếc đã có lỗi xảy ra
Dữ liệu đầu vào không hợp lệ
```

Runtime local đã hoạt động nhưng cần xử lý test configuration và cập nhật expectation trước khi kết
luận migration hoàn tất.

## 7. Thành phần còn thuộc OCR

Các thành phần sau vẫn nằm tại service vì mang tính nghiệp vụ hoặc provider-specific:

- FPT OCR/eKYC contracts và endpoint mapping.
- Domain entity.
- Business service orchestration.
- Liquibase schema và migration.
- Kafka topic nghiệp vụ.
- Media processing workflow.
- Hai encryption key riêng cho payload và response audit.
- Static OpenAPI document của eKYC BFF.

## 8. Kết luận

Việc áp dụng common đã loại bỏ phần lớn infrastructure duplicate khỏi OCR:

- Giảm 38,4% số production Java file.
- Giảm 27,1% Java production LOC.
- Giảm 52,9% direct Maven dependency.
- Giảm 52,1% cấu hình `application.yml`.
- Chuẩn hóa API response, exception, repository, entity base, security, HTTP client, logging, Kafka
  và health check.

Sau migration, source OCR tập trung nhiều hơn vào nghiệp vụ thực tế. Những thay đổi hiện tại chưa
được commit; full test suite cần được làm xanh trước khi merge hoặc release.
