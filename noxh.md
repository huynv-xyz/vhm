# Vấn đề 6: Lựa chọn nền tảng tích hợp OCR tập trung

## 1. Phạm vi

Tài liệu này chỉ thiết kế **luồng OCR tích hợp bằng backend API**. Mobile/Web không tích hợp SDK của FPT và không gọi trực tiếp FPT AI.

Phạm vi bao gồm:

- Upload một tài liệu lên `vhm-media-service` bằng Presigned PUT.
- Tạo yêu cầu OCR qua `vhm-agent-api` và `vhm-dossier-core`.
- `vhm-verification-service` xử lý OCR bất đồng bộ bằng queue/worker.
- Worker gọi FPT AI bằng server credential, chuẩn hóa và lưu kết quả.
- Mobile/Web polling trạng thái/kết quả từ API của VHM và xác nhận dữ liệu trước khi apply vào hồ sơ.

Phạm vi hiện tại **không bao gồm** eKYC, FPT SDK, liveness, face matching, NFC hoặc SDK Proxy. Các năng lực này sẽ được thiết kế riêng khi chốt phương án SDK.

Các tài liệu nghiệp vụ dự kiến:

- CCCD/CMND khi API/model được chọn hỗ trợ đúng input contract.
- Giấy đăng ký kết hôn.
- Bản sao có chứng thực giấy chứng nhận hộ nghèo/cận nghèo.
- Các giấy tờ khác trong checklist hồ sơ NOXH sau khi có model/template và schema kết quả tương ứng.

Mỗi yêu cầu OCR xử lý **một logical document**, được tham chiếu bằng `s3PathFile` do `vhm-media-service` sinh. `documentType` chỉ dùng để chọn provider API/model và schema chuẩn hóa; nó không tạo một workflow khác.

OCR chỉ hỗ trợ số hóa và gợi ý dữ liệu. Kết quả OCR không xác minh người thao tác là chủ thể của giấy tờ và không tự quyết định hồ sơ đủ điều kiện NOXH. Người dùng phải xác nhận trước khi `vhm-dossier-core` cập nhật dữ liệu nghiệp vụ.

## 2. Quyết định kiến trúc

| **Hướng tiếp cận** | **Ưu điểm** | **Nhược điểm** | **Lựa chọn** |
| --- | --- | --- | --- |
| `vhm-dossier-core` gọi trực tiếp FPT AI | Ít thành phần ban đầu | Domain NOXH phụ thuộc credential, session, payload, error code và SLA của FPT; khó tái sử dụng cho domain khác | No |
| `vhm-verification-service` tích hợp FPT AI | Tập trung provider adapter, credential, queue/worker, retry, timeout và Canonical Result; domain chỉ authorize và apply | Thêm một service và hạ tầng job | **Yes** |

`vhm-verification-service` cung cấp contract OCR ổn định cho các domain. FPT `/ocr` là synchronous, nhưng API của VHM vẫn là asynchronous:

```text
Mobile/Web → VHM create OCR → 202 QUEUED
OCR Worker → FPT /session/init → FPT /ocr synchronous
OCR Worker → lưu Canonical Result
Mobile/Web → poll VHM status/result
```

Cách bọc này tránh giữ kết nối Mobile/Web trong thời gian upload media từ storage và chờ provider, đồng thời cô lập timeout/backlog của FPT khỏi `vhm-dossier-core`.

## 3. Kiến trúc OCR tập trung

### 3.1. Thành phần

| **Thành phần** | **Trách nhiệm** |
| --- | --- |
| Mobile/Web | Upload tài liệu; tạo OCR; poll; hiển thị và cho người dùng xác nhận kết quả |
| `vhm-agent-api` | Xác thực/routing và authorize yêu cầu cấp Presigned PUT; không giữ FPT credential |
| `vhm-dossier-core` | Authorize hồ sơ, `businessRef`, yêu cầu OCR, query/apply kết quả |
| `vhm-media-service` | Chỉ tham gia upload: trả `presignHeaders + presignedUrl + s3PathFile` |
| Verification API | Private command/query API của `vhm-verification-service` |
| OCR Worker | Claim job, đọc tài liệu, gọi provider và persist kết quả; không mở public API |
| Provider Adapter | Ánh xạ `documentType` sang FPT API/model; inject credential; map response/error |
| Result Normalizer | Chuyển provider response thành Canonical Result có version |
| Verification Database + Outbox | Lưu lifecycle, `s3PathFile`, worker lease/retry, provider attempt, result và sự kiện |

`Verification API`, `OCR Worker`, `Provider Adapter` và `Result Normalizer` là module/workload trong cùng `vhm-verification-service`, không phải các service public riêng.

### 3.2. Thành phần và hướng dữ liệu

```mermaid
flowchart TB
    CLIENT["`**Mobile/Web**
Upload · Submit OCR · Poll`"]
    BFF["`**vhm-agent-api**
Xác thực · routing`"]
    DOSSIER["`**vhm-dossier-core**
Authorize · Apply result`"]
    MEDIA_API["`**vhm-media-service**
Presigned PUT`"]
    STORAGE[("`**Private Object Storage**
OCR document`")]
    subgraph VERIFY["`**vhm-verification-service (private)**
OCR · Provider Adapter`"]
        direction LR
        API["`**Verification API**
Create · Status · Result`"]
        OUTBOX["`**Outbox Publisher**
Publish OCR job`"]
        WORKER["`**OCR Worker**
Internal workload`"]
        ADAPTER["`**Provider Adapter**
FPT contract · credential`"]

        WORKER --> ADAPTER
    end
    QUEUE[("`**OCR Job Queue**
Async job`")]
    FPT["`**FPT AI Backend**
Session API · OCR API`"]
    DB[("`**Verification Database**
Verification · Provider attempt · Result`")]

    CLIENT <-->|"Application API"| BFF
    BFF <-->|"Request upload URL"| MEDIA_API
    CLIENT ==>|"Presigned PUT"| STORAGE
    MEDIA_API -->|"Sign PUT for configured bucket/prefix"| STORAGE

    BFF <-->|"Create/query/apply OCR"| DOSSIER
    DOSSIER <-->|"Private OCR command/query"| API

    API <-->|"Persist/read status · result"| DB
    OUTBOX <-->|"Claim committed outbox"| DB
    OUTBOX -->|"Publish"| QUEUE
    QUEUE -->|"Worker consume"| WORKER

    WORKER -->|"HEAD/GET basePrefix + s3PathFile<br/>read-only IAM"| STORAGE
    ADAPTER <-->|"Synchronous provider API"| FPT
    WORKER -->|"Persist attempt · result"| DB
```

Hai đường dữ liệu được tách rõ:

- Upload binary: `Mobile/Web → Presigned PUT → Object Storage`.
- OCR: `vhm-verification-service → Object Storage → FPT AI Backend`.

Mobile/Web không gửi binary qua `vhm-dossier-core` hoặc `vhm-verification-service` khi upload và không gọi trực tiếp FPT. Khi tạo OCR, Verification API persist `QUEUED` cùng outbox; Outbox Publisher đưa job vào `OCR Job Queue`, sau đó OCR Worker mới đọc media và gọi FPT. Các module nằm trong khung `vhm-verification-service` thuộc cùng một service/codebase; queue là hạ tầng messaging của workload này.

### 3.3. Lifecycle và outcome

`status` mô tả vòng đời kỹ thuật:

```text
QUEUED → PROCESSING → COMPLETED
   └───────────────→ CANCELLED/EXPIRED
```

- `QUEUED`: verification đã lưu `s3PathFile`/worker state và có outbox để publish message.
- `PROCESSING`: worker đã claim job.
- `COMPLETED`: đã có kết luận cuối cùng, kể cả trường hợp provider lỗi sau khi hết recovery budget.
- `CANCELLED/EXPIRED`: kết thúc không có outcome.

`outcome` chỉ có khi `status=COMPLETED`:

| **Outcome** | **Ý nghĩa** |
| --- | --- |
| `OCR_COMPLETED` | Đã đọc và chuẩn hóa tài liệu; người dùng vẫn phải xác nhận |
| `NEED_REVIEW` | Kết quả có confidence/warning cần kiểm tra thủ công |
| `NEED_RETRY` | Input không đọc được hoặc chất lượng không đạt; cần upload tài liệu mới |
| `PROVIDER_ERROR` | Hết recovery budget hoặc provider không trả kết quả hợp lệ |

Lỗi transport/provider không được ánh xạ thành lỗi nghiệp vụ của hồ sơ.

## 4. Upload tài liệu

Upload diễn ra trước OCR và không tạo `verificationId`. `vhm-agent-api` gọi thẳng `vhm-media-service`; `vhm-dossier-core` và `vhm-verification-service` không nằm trong luồng upload. Contract hiện có không trả `mediaId` và không có bước finalize trong flow này.

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile/Web
    participant BFF as vhm-agent-api
    participant MEDIA_API as vhm-media-service
    participant STORAGE as Private Object Storage

    CLIENT->>BFF: Request Presigned PUT<br/>businessRef + documentType + file metadata
    BFF->>BFF: Authenticate + authorize upload
    BFF->>MEDIA_API: Request Presigned PUT<br/>authorized path context + file metadata
    MEDIA_API-->>BFF: code + message + data<br/>presignHeaders + presignedUrl + s3PathFile
    BFF-->>CLIENT: presignHeaders + presignedUrl + s3PathFile

    CLIENT->>STORAGE: PUT binary<br/>Content-Type + toàn bộ presignHeaders
    STORAGE-->>CLIENT: Upload response
```

Response hiện có của Media Service:

```json
{
  "code": 0,
  "message": "Thành công",
  "data": {
    "presignHeaders": {
      "x-amz-meta-vhm_performer": "dossier-svc",
      "x-amz-meta-vhm_performer_source": "http-internal",
      "x-amz-server-side-encryption": "aws:kms",
      "x-amz-server-side-encryption-aws-kms-key-id": "<kms-key-arn>"
    },
    "presignedUrl": "<temporary-presigned-put-url>",
    "s3PathFile": "registrations/9170c0b1-09ce-4016-b701-3626386abea0/document-c9eedd4c.png"
  }
}
```

Quy ước tích hợp:

- Client phải gửi đúng `Content-Type` và toàn bộ `presignHeaders`; thay đổi hoặc thiếu signed header làm S3 từ chối request.
- Sau khi S3 trả upload thành công, Mobile/Web submit OCR bằng `s3PathFile`; không có request finalize riêng.
- Chỉ `s3PathFile` được dùng làm durable reference. Không persist hoặc chuyển tiếp lại `presignedUrl` sau upload vì URL chứa credential tạm thời và sẽ hết hạn.
- `s3PathFile` phải là relative path do Media Service sinh trong business prefix đã authorize; không nhận full S3 URL, bucket hoặc path tùy ý từ client.
- Trước khi gọi FPT, worker chuẩn hóa `s3PathFile`, ghép với bucket/base prefix cấu hình, rồi HEAD/GET S3 bằng IAM read-only. Worker kiểm tra object tồn tại cùng MIME/magic bytes/kích thước trước khi đọc binary.
- Contract hiện tại không cung cấp `mediaId`, trạng thái `FINALIZED` hoặc bằng chứng object immutable; TDD không giả định các năng lực này.

## 5. Luồng OCR bằng API

### 5.1. Luồng tổng quan trong `vhm-verification-service`

```mermaid
flowchart LR
    API["`**Verification API**
Validate request`"]
    ACCEPT["`**Database transaction**
QUEUED · s3PathFile · Outbox`"]
    QUEUE[("`**OCR Job Queue**`")]
    WORKER["`**OCR Worker**
PROCESSING`"]
    READ["`**Storage Reader**
Normalize path · HEAD/GET S3`"]
    INIT["`**FPT Session API**
POST /session/init`"]
    OCR["`**FPT OCR API**
POST /ocr · synchronous`"]
    NORMALIZE["`**Result Normalizer**
Canonical fields · warnings`"]
    DONE["`**Verification Database**
COMPLETED · outcome · result`"]
    STATUS["`**Verification API**
GET status/result`"]

    API --> ACCEPT -->|"Publish"| QUEUE --> WORKER
    WORKER --> READ --> INIT --> OCR --> NORMALIZE --> DONE
    DONE --> STATUS
```

Toàn bộ box từ `Verification API` đến `Result Normalizer` thuộc cùng `vhm-verification-service`. FPT không xử lý job bất đồng bộ trong contract `/ocr` đang dùng; worker chờ response synchronous rồi mới normalize và persist.

### 5.2. Sequence diagram end-to-end

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile/Web
    participant BFF as vhm-agent-api
    participant DOSSIER as vhm-dossier-core
    participant VERIFY as vhm-verification-service
    participant DB as Verification Database
    participant QUEUE as OCR Job Queue
    participant STORAGE as Private Object Storage
    participant FPT as FPT AI Backend

    CLIENT->>BFF: Submit OCR<br/>dossierId + s3PathFile + documentType
    BFF->>DOSSIER: Authenticated request
    DOSSIER->>DOSSIER: Authorize dossier + OCR action<br/>validate registration path prefix
    DOSSIER->>VERIFY: POST OCR<br/>businessRef + s3PathFile + documentType
    VERIFY->>DB: Transaction: verification QUEUED<br/>s3PathFile + worker state + outbox
    DB-->>VERIFY: Commit
    VERIFY-->>DOSSIER: 202 + verificationId + resourceUri
    DOSSIER-->>BFF: 202 + verificationId + statusUrl
    BFF-->>CLIENT: 202 + verificationId + statusUrl

    DB-->>QUEUE: Outbox publisher: OCR_JOB_CREATED
    QUEUE-->>VERIFY: Worker claim verificationId
    VERIFY->>DB: Claim lease + verification PROCESSING

    VERIFY->>VERIFY: Normalize s3PathFile<br/>fullKey = basePrefix + s3PathFile
    VERIFY->>STORAGE: HEAD object bằng read-only IAM
    STORAGE-->>VERIFY: Object metadata
    VERIFY->>STORAGE: GET document stream bằng read-only IAM
    STORAGE-->>VERIFY: Binary
    VERIFY->>VERIFY: Validate MIME/magic bytes + size

    VERIFY->>FPT: POST /session/init<br/>api-key + client_uuid=verificationId + only-engine=1
    FPT-->>VERIFY: 200 + session-id
    VERIFY->>FPT: POST /ocr<br/>session-id + document-type + file
    FPT-->>VERIFY: Synchronous errorCode + data

    VERIFY->>VERIFY: Map provider error + normalize result
    VERIFY->>DB: Transaction: provider attempt + result<br/>COMPLETED + outcome + outbox

    loop Khi status chưa kết thúc
        CLIENT->>BFF: GET statusUrl
        BFF->>DOSSIER: Authorized status query
        DOSSIER->>VERIFY: GET verificationId
        VERIFY->>DB: Read status/result
        DB-->>VERIFY: Current state
        VERIFY-->>DOSSIER: Status + outcome/result
        DOSSIER-->>BFF: Authorized projection
        BFF-->>CLIENT: Status + outcome/result
    end

    CLIENT->>BFF: Confirm OCR result<br/>verificationId
    BFF->>DOSSIER: Apply confirmed OCR result
    DOSSIER->>DOSSIER: Update dossier in local transaction
```

Hai call FPT có vai trò khác nhau:

- `/session/init` tạo phiên provider và trả `session-id` bắt buộc cho `/ocr`.
- `/ocr` nhận document và trả kết quả OCR đồng bộ.

VHM không trả `session-id` xuống Mobile/Web và không dùng nó làm `verificationId`.

## 6. API contract của `vhm-verification-service`

### 6.1. Danh sách API

| **Use case** | **Private API** | **Response** |
| --- | --- | --- |
| Tạo OCR | `POST /v1/ocr-verifications` | `202 + verificationId + resourceUri` |
| Lấy trạng thái | `GET /v1/ocr-verifications/{verificationId}` | Status, outcome, next action |
| Lấy kết quả | `GET /v1/ocr-verifications/{verificationId}/result` | Canonical Result |
| Thử lại | `POST /v1/ocr-verifications/{verificationId}/retries` | `202`, tạo verification mới |

Các API trên chỉ được expose qua private ingress/DNS cho service caller có mTLS hoặc service token đúng scope. Prefix `/internal` không được sử dụng vì toàn bộ API của `vhm-verification-service` đã là private. URL không chứa khái niệm `dossierId`; domain caller truyền định danh nghiệp vụ bằng `businessRef.type + businessRef.id` trong command và tự lưu association với `verificationId`.

Các route `/dossiers/...` nếu có thuộc Application API của `vhm-agent-api`/`vhm-dossier-core`, không phải contract của service dùng chung. Tương tự, cấp Presigned PUT thuộc API hiện có của `vhm-media-service`, còn confirm/apply kết quả thuộc API và transaction của từng domain; các nhóm API này không được định nghĩa lại trong mục này.

`vhm-verification-service` trả service `resourceUri`. Domain caller ánh xạ URI này thành URL thuộc application của mình và authorize lại trên mỗi request từ Mobile/Web; URI của private service không được chuyển nguyên cho client.

### 6.2. Tạo OCR

```http
POST /v1/ocr-verifications
Authorization: Bearer <service-token>
Idempotency-Key: 72aacfa4-97b8-4d0f-bb74-490f17352b1b
Content-Type: application/json

{
  "businessRef": { "type": "DOSSIER", "id": "dos-01" },
  "subjectRef": "customer-opaque-ref",
  "channel": "MOBILE",
  "documentType": "MARRIAGE_CERTIFICATE",
  "s3PathFile": "registrations/9170c0b1-09ce-4016-b701-3626386abea0/document-c9eedd4c.png"
}
```

```http
HTTP/1.1 202 Accepted
Retry-After: 3

{
  "verificationId": "ver-123",
  "status": "QUEUED",
  "resourceUri": "/v1/ocr-verifications/ver-123"
}
```

Create chỉ trả `202` sau khi verification chứa `s3PathFile`/worker state và outbox đã commit thành công. Nếu không thể persist an toàn, API trả lỗi; không trả `202` rồi để mất yêu cầu xử lý.

### 6.3. Status và result

```json
{
  "verificationId": "ver-123",
  "status": "PROCESSING",
  "outcome": null,
  "resultAvailable": false,
  "nextAction": "POLL",
  "updatedAt": "2026-08-10T11:30:25+07:00"
}
```

Canonical Result không phụ thuộc tên field/error code của FPT:

```json
{
  "verificationId": "ver-123",
  "status": "COMPLETED",
  "outcome": "OCR_COMPLETED",
  "schemaVersion": "1.0",
  "document": {
    "type": "MARRIAGE_CERTIFICATE",
    "fields": {
      "certificateNumber": {
        "value": "01/2026",
        "confidence": 0.97
      },
      "registrationDate": {
        "value": "2026-08-01",
        "confidence": 0.95
      }
    },
    "warnings": []
  },
  "nextAction": "CONFIRM_AND_APPLY",
  "completedAt": "2026-08-10T11:31:12+07:00"
}
```

Kết quả chỉ chứa field nằm trong allowlist của `documentType`. Raw provider payload, API key, provider session, storage path và read-grant URL không được trả qua Application API.

### 6.4. HTTP và error contract

| **HTTP** | **Sử dụng** |
| --- | --- |
| `200/201/202` | Query/create thành công hoặc command đã được persist để xử lý |
| `400` | Header/payload sai định dạng |
| `401/403` | Service identity hoặc scope không hợp lệ |
| `404` | Resource không tồn tại hoặc caller không được phép biết resource tồn tại |
| `409` | Idempotency conflict, state transition sai hoặc result chưa sẵn sàng |
| `413/415/422` | Media vượt giới hạn, sai loại hoặc không đúng contract của `documentType` |
| `429` | Admission control/quota nội bộ; trả `Retry-After` |
| `503` | Không thể persist/enqueue an toàn |

Error response của VHM chỉ dùng `code`, `message`, `retryable` và `correlationId`. Provider error phát sinh trong worker được lưu vào verification/provider attempt để Mobile/Web nhận khi poll, không trả raw FPT response.

## 7. Dữ liệu

### 7.1. Schema logic

| **Bảng** | **Mục đích** |
| --- | --- |
| `ocr_verifications` | Aggregate root; chứa business binding, `s3PathFile`, lifecycle, idempotency và worker lease/retry |
| `provider_attempts` | Ghi từng lần init/OCR và trạng thái timeout không xác định |
| `ocr_results` | Canonical Result hiện hành, có version |
| `outbox_events` | Bảo đảm DB commit và enqueue/event không lệch nhau |

Baseline quy ước một verification có một `s3PathFile` và một workload OCR. Verification Database chỉ lưu relative path do `vhm-media-service` sinh; không lưu Presigned URL, bucket hoặc AWS credential. Worker dùng bucket/base prefix cấu hình và IAM read-only để HEAD/GET trực tiếp S3 trước khi gọi FPT. Chỉ tách bảng file reference khi một verification thực sự cần nhiều tài liệu.

```mermaid
erDiagram
    OCR_VERIFICATION ||--o{ PROVIDER_ATTEMPT : invokes
    OCR_VERIFICATION ||--o| OCR_RESULT : produces
    OCR_VERIFICATION ||--o{ OUTBOX_EVENT : publishes
```

### 7.2. DDL baseline rút gọn

```sql
CREATE TABLE ocr_verifications (
    verification_id         UUID PRIMARY KEY,
    business_type           VARCHAR(30) NOT NULL,
    business_ref            VARCHAR(100) NOT NULL,
    subject_ref_ciphertext  BYTEA,
    channel                 VARCHAR(20) NOT NULL CHECK (channel IN ('MOBILE', 'WEB')),
    document_type           VARCHAR(50) NOT NULL,

    s3_path_file            TEXT NOT NULL,

    status                  VARCHAR(30) NOT NULL CHECK (status IN
                                ('QUEUED', 'PROCESSING', 'COMPLETED',
                                 'CANCELLED', 'EXPIRED')),
    outcome                 VARCHAR(30),
    attempt_count           INTEGER NOT NULL DEFAULT 0 CHECK (attempt_count >= 0),
    available_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    lease_owner             VARCHAR(100),
    lease_until             TIMESTAMPTZ,
    last_error_code         VARCHAR(80),

    retry_of                UUID REFERENCES ocr_verifications(verification_id),
    idempotency_key         VARCHAR(100) NOT NULL,
    request_fingerprint     CHAR(64) NOT NULL,
    terminal_reason_code    VARCHAR(80),
    row_version             BIGINT NOT NULL DEFAULT 0,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at            TIMESTAMPTZ,
    CONSTRAINT uq_ocr_idempotency UNIQUE (business_type, idempotency_key),
    CONSTRAINT ck_ocr_status_outcome CHECK (
        (status = 'COMPLETED' AND outcome IN
            ('OCR_COMPLETED', 'NEED_REVIEW', 'NEED_RETRY', 'PROVIDER_ERROR')) OR
        (status <> 'COMPLETED' AND outcome IS NULL)
    ),
    CONSTRAINT ck_ocr_completed_at CHECK (
        (status = 'COMPLETED' AND completed_at IS NOT NULL) OR
        (status <> 'COMPLETED' AND completed_at IS NULL)
    )
);

CREATE INDEX ix_ocr_business
    ON ocr_verifications (business_type, business_ref, created_at DESC);
CREATE INDEX ix_ocr_dispatch
    ON ocr_verifications (status, available_at)
    WHERE status IN ('QUEUED', 'PROCESSING');

CREATE TABLE provider_attempts (
    provider_attempt_id      UUID PRIMARY KEY,
    verification_id         UUID NOT NULL REFERENCES ocr_verifications(verification_id),
    provider                 VARCHAR(30) NOT NULL,
    operation                VARCHAR(30) NOT NULL CHECK (operation IN ('INIT_SESSION', 'OCR')),
    attempt_no               INTEGER NOT NULL CHECK (attempt_no > 0),
    provider_session_ciphertext BYTEA,
    status                   VARCHAR(20) NOT NULL CHECK (status IN
                                ('STARTED', 'SUCCEEDED', 'FAILED', 'UNKNOWN')),
    delivery_state           VARCHAR(20) NOT NULL CHECK (delivery_state IN
                                ('NOT_SENT', 'SENDING', 'SENT', 'UNKNOWN')),
    error_class              VARCHAR(40),
    error_code               VARCHAR(80),
    started_at               TIMESTAMPTZ NOT NULL,
    finished_at              TIMESTAMPTZ,
    CONSTRAINT uq_provider_call_attempt
        UNIQUE (verification_id, provider, operation, attempt_no)
);

CREATE TABLE ocr_results (
    verification_id         UUID PRIMARY KEY REFERENCES ocr_verifications(verification_id),
    schema_version           VARCHAR(20) NOT NULL,
    canonical_payload_ciphertext BYTEA NOT NULL,
    payload_key_version      VARCHAR(40) NOT NULL,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE outbox_events (
    event_id                 UUID PRIMARY KEY,
    aggregate_id             UUID NOT NULL,
    event_type               VARCHAR(80) NOT NULL,
    payload                  JSONB NOT NULL,
    status                   VARCHAR(20) NOT NULL DEFAULT 'NEW' CHECK (status IN
                                ('NEW', 'PUBLISHING', 'PUBLISHED', 'FAILED')),
    available_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    lease_owner              VARCHAR(100),
    lease_until              TIMESTAMPTZ,
    published_at             TIMESTAMPTZ,
    attempt_count            INTEGER NOT NULL DEFAULT 0,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT ck_outbox_publishing_lease CHECK (
        (status = 'PUBLISHING' AND lease_owner IS NOT NULL AND lease_until IS NOT NULL) OR
        (status <> 'PUBLISHING' AND lease_owner IS NULL AND lease_until IS NULL)
    )
);

CREATE INDEX ix_outbox_publish
    ON outbox_events (available_at)
    WHERE status IN ('NEW', 'FAILED');
CREATE INDEX ix_outbox_lease_recovery
    ON outbox_events (lease_until)
    WHERE status = 'PUBLISHING';
```

Chỉ lưu relative `s3PathFile`; không lưu bucket, full object key, Presigned URL hoặc raw provider payload. Outbox chỉ chứa ID/reference tối thiểu, không chứa tài liệu hoặc Canonical Result.

Retry budget được lấy từ cấu hình theo provider/operation, không lưu `max_attempts` trên từng verification. Schema version chỉ lưu cùng Canonical Result trong `ocr_results.schema_version`. `nextAction` trong API response được tính từ `status + outcome`, không persist trong database. Mỗi verification có một result immutable nên không cần `result_version` hoặc `result_hash`; retry tạo `verificationId` mới.

## 8. Xử lý tin cậy và timeout

### 8.1. Transaction và idempotency

- Domain caller chỉ gửi `s3PathFile` đã nhận từ upload flow và đã kiểm tra business prefix; Verification API validate đây là relative path hợp lệ, không nhận full URL/bucket.
- Sau đó insert `ocr_verifications` chứa `s3PathFile`/worker state và `outbox_events` trong một transaction ngắn.
- Cùng `Idempotency-Key` và fingerprint trả resource cũ; cùng key nhưng fingerprint khác trả `409 IDEMPOTENCY_CONFLICT`.
- Queue dùng at-least-once. Message chỉ chứa `verificationId`; worker claim aggregate bằng lease/CAS và không gọi provider lại nếu đã có provider attempt terminal thành công.
- Outbox Publisher claim event `NEW/FAILED` bằng `PUBLISHING + lease`; khi thành công chuyển `PUBLISHED`, khi lỗi chuyển `FAILED` và đặt `available_at` theo backoff. Event `PUBLISHING` có lease hết hạn được publisher khác phục hồi, vì vậy không bị kẹt khi process chết giữa chừng.
- Provider call và Object Storage HEAD/GET luôn nằm ngoài DB transaction.
- Persist result trong transaction ngắn: provider attempt, Canonical Result, lifecycle và outbox `OCR_COMPLETED`.

### 8.2. Timeout FPT synchronous API

Timeout có thể xảy ra tại kết nối, gateway hoặc trong lúc FPT xử lý. Timeout chỉ có nghĩa worker không nhận response trước deadline; nó không chứng minh FPT chưa xử lý document.

| **Thời điểm lỗi** | **Xử lý** |
| --- | --- |
| Chưa gửi request/body tới FPT | Có thể retry có giới hạn với backoff |
| FPT trả `429/5xx` rõ ràng | Retry khi contract/SLA xác nhận operation retry-safe |
| Timeout sau khi body có thể đã gửi | Ghi `status=UNKNOWN`, `delivery_state=UNKNOWN`; không POST `/ocr` lại mù |
| FPT trả business OCR error | Map sang `NEED_RETRY` hoặc `NEED_REVIEW`; không coi là timeout |

Normal flow không polling FPT vì `/ocr` trả kết quả synchronous. Nếu timeout sau khi body có thể đã được gửi, worker không gọi lại `/ocr` một cách mù quáng; verification kết thúc `COMPLETED + PROVIDER_ERROR`, `nextAction=RETRY`. Retry từ người dùng tạo `verificationId` mới, không reset hoặc ghi đè attempt cũ.

Timeout outbound tới FPT phải ngắn hơn lease của worker. Giá trị cụ thể cần được chốt theo SLA và p95/p99 thực tế của từng endpoint; tài liệu này không giả định một con số timeout là SLA chính thức của FPT.

### 8.3. Kết thúc xử lý

- Thành công: verification chuyển `COMPLETED` với outcome `OCR_COMPLETED/NEED_REVIEW/NEED_RETRY`, đồng thời xóa worker lease.
- Lỗi retry-safe: verification trở lại `QUEUED`, tăng `attempt_count` và đặt `available_at` theo backoff.
- Hết recovery budget: verification chuyển `COMPLETED + PROVIDER_ERROR`; client không bị poll vô hạn.

## 9. Kế hoạch triển khai và kiểm thử

### 9.1. Thứ tự triển khai

1. Chốt bucket/base prefix, quy tắc `s3PathFile` và IAM read-only `HeadObject/GetObject` cho OCR Worker.
2. Chốt với FPT từng `documentType → API/model`, input contract, file limit, response/error, quota và timeout.
3. Xây Verification API, schema, idempotency và outbox/queue/worker skeleton bằng mock Provider Adapter.
4. Tích hợp FPT `/session/init → /ocr`, Result Normalizer và polling end-to-end.
5. Chỉ enable document type đã có provider contract và fixture được xác nhận.
6. Load/resilience test; chốt concurrency, retry/recovery budget và alert backlog trước production.

### 9.2. Kiểm thử tối thiểu

| **Lớp test** | **Phạm vi** |
| --- | --- |
| Unit | Lifecycle/outcome guard, idempotency fingerprint, field mapping, confidence/warning và timeout classification |
| Contract | Media upload response; FPT init/OCR success, business error, malformed response, 429, 5xx và timeout fixture |
| Integration | PostgreSQL constraint/locking, outbox publish, queue duplicate, worker lease/recovery và authorized `s3PathFile` |
| End-to-end | Presigned PUT → create `202` bằng `s3PathFile` → processing → result → confirm/apply |
| Resilience | FPT slow/timeout, unknown-after-send, queue backlog, worker crash, DLQ và client polling không vô hạn |

### 9.3. Điểm cần chốt

- API/model FPT cho giấy đăng ký kết hôn, giấy chứng nhận hộ nghèo/cận nghèo và tài liệu ngoài `idr/passport/dlr`.
- File type/size/page limit, request multipart contract và SLA/quota của từng model.
- Threshold confidence/warning theo từng `documentType`.
- Retention của Canonical Result, provider session metadata và centralized audit log.

`vhm-verification-service` chịu trách nhiệm OCR và cô lập FPT contract. `vhm-dossier-core` vẫn chịu trách nhiệm authorization, xác nhận người dùng và cập nhật hồ sơ NOXH.
