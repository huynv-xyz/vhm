# Vấn đề 6: Thiết kế tích hợp OCR và eKYC tập trung

## 1. Phạm vi và quyết định kiến trúc

`vhm-verification-service` cung cấp capability OCR/eKYC tập trung cho nhiều domain. Mobile/Web không gọi trực tiếp FPT AI; domain service không phụ thuộc credential, session, payload hoặc error code của provider.

### 1.1. Phương án được chọn

Baseline dùng hai execution path tách biệt theo capability:

```text
DOCUMENT_OCR:
Upload document → create OCR → 202 QUEUED
→ OCR Worker → FPT session/init → OCR
→ Canonical Result → Mobile/Web poll → confirm/apply

IDENTITY_EKYC:
Mobile tự cấu hình flow, client_uuid, Proxy Base URL và custom VHM auth header cho FPT SDK
→ SDK tự điều phối init session → OCR → liveness
→ mỗi request đi qua vhm-verification-service → FPT AI Backend
→ vhm-verification-service audit response và forward nguyên response về SDK
→ SDK trả kết quả cho Mobile
```

OCR tài liệu nghiệp vụ vẫn dùng **backend API + outbox/queue/worker**. Chỉ eKYC định danh dùng **FPT SDK qua `vhm-verification-service`** để tận dụng capture UX, kiểm tra chất lượng đầu vào, liveness và device capability của SDK nhưng không cho SDK gọi trực tiếp FPT.

Các eKYC SDK endpoint synchronous của `vhm-verification-service` nằm trên critical path của SDK; request init/OCR định danh/liveness không đi qua Queue hoặc eKYC Worker. Mobile tự chọn và cấu hình SDK flow. Service chỉ xác thực request, inject FPT credential, forward request/response và audit response; không cấp SDK config, không tạo FPT session và không điều phối step.

### 1.2. So sánh các hướng tích hợp FPT

FPT SDK là thư viện chạy trên ứng dụng Mobile để hướng dẫn capture và điều phối các bước eKYC. SDK không chạy trong Verification API hoặc worker. Nếu chọn SDK, kiến trúc client và đường truyền media phải thay đổi tương ứng.

| **Hướng tiếp cận** | **Luồng chính** | **Ưu điểm** | **Nhược điểm** | **Lựa chọn** |
| --- | --- | --- | --- | --- |
| Backend API tại từng domain | Mobile/Web gọi domain backend; mỗi domain tự quản lý media, session và gọi FPT API | Không cần thêm capability/service tập trung; domain chủ động tùy chỉnh luồng theo nghiệp vụ riêng | Lặp code tích hợp ở nhiều domain; phân tán credential, session, retry, quota và audit; kết quả/error contract dễ không đồng nhất; mỗi lần đổi provider phải sửa nhiều service | No |
| Backend API qua `vhm-verification-service` | Mobile/Web upload vào S3; domain gọi Verification API; Queue phân phối job để worker đọc media và gọi FPT API | Phù hợp tài liệu nghiệp vụ, file lớn và xử lý nền; domain không tự quản provider credential/session/retry; latency, quota và backlog được cô lập | Phải vận hành database, queue, Outbox Publisher và worker; VHM tự xây upload/capture UX | **Yes — chỉ `DOCUMENT_OCR`** |
| FPT SDK gọi trực tiếp FPT | SDK trên Mobile capture và gửi dữ liệu thẳng tới FPT; VHM nhận kết quả theo cơ chế tích hợp của FPT | Tích hợp eKYC phía VHM đơn giản hơn; ít hop và độ trễ truyền tải thấp; tận dụng capture UI, kiểm tra chất lượng thời gian thực, liveness và các capability SDK hỗ trợ | VHM không kiểm soát đường truyền media trước khi tới FPT; phụ thuộc SDK trên từng nền tảng và cơ chế nhận/đối soát kết quả; khó áp dụng contract upload/worker hiện tại; không phù hợp OCR tài liệu nghiệp vụ tổng quát | No |
| FPT SDK qua `vhm-verification-service` | Mobile tự cấu hình SDK gọi eKYC endpoint của `vhm-verification-service`; service inject credential, forward request/response với FPT và audit response | Tận dụng capture UX và kiểm tra chất lượng của SDK; VHM kiểm soát data flow và có dữ liệu audit; Mobile không giữ FPT credential | `vhm-verification-service` nằm trên synchronous streaming path nên phải HA, kiểm soát latency/bandwidth và tương thích chặt SDK protocol; không dùng queue để che lỗi trên interactive path | **Yes — chỉ `IDENTITY_EKYC`** |

Không chọn backend riêng tại từng domain vì phần tích hợp FPT là capability kỹ thuật lặp lại, không phải nghiệp vụ riêng của từng domain. `vhm-verification-service` quản lý provider contract và credential; eKYC response đi qua service chỉ được lưu phục vụ audit, chưa normalize hoặc apply vào nghiệp vụ trong baseline này.

Quyết định hybrid không dùng SDK cho OCR tài liệu nghiệp vụ. PDF/tài liệu dung lượng lớn đi qua S3 và OCR Worker; các eKYC SDK endpoint của `vhm-verification-service` chỉ nhận media định danh và liveness theo giới hạn contract FPT. Hai execution path phải có timeout, quota, capacity guard và observability độc lập.

OCR chỉ hỗ trợ số hóa/gợi ý dữ liệu. eKYC hỗ trợ kiểm tra giấy tờ, liveness và face match. Cả hai không tự quyết định hồ sơ đủ điều kiện NOXH; người dùng phải xác nhận trước khi domain apply kết quả.

## 2. Thành phần và quy ước nền tảng

### 2.1. Thành phần và trách nhiệm

| **Thành phần** | **Trách nhiệm** |
| --- | --- |
| Mobile/Web | OCR: upload tài liệu và poll; eKYC chỉ trên Mobile: cấu hình flow, client UUID, Proxy Base URL và custom VHM auth header, khởi chạy SDK và nhận kết quả SDK |
| FPT eKYC SDK | Capture giấy tờ định danh/liveness, kiểm tra chất lượng trên thiết bị và gọi đúng eKYC endpoint của `vhm-verification-service` |
| `vhm-agent-api` | Xác thực/routing; authorize upload OCR và publish streaming route eKYC; không giữ FPT credential |
| Domain service, ví dụ `vhm-dossier-core` | Authorize `businessRef`, media path và apply kết quả OCR |
| `vhm-media-service` | Chỉ tham gia upload OCR tài liệu; trả `presignHeaders + presignedUrl + s3PathFile` |
| Verification API | OCR command/query và các endpoint forward request từ FPT SDK |
| Outbox Publisher + Job Queue | Chỉ dispatch OCR job và event hậu xử lý sau commit; không dùng cho eKYC SDK call |
| OCR Worker pool | Xử lý một logical document và gọi FPT OCR flow |
| Provider Adapter | OCR Worker: map provider API; eKYC SDK endpoint: inject credential và giữ wire contract tương thích SDK |
| Result Normalizer/Policy | Tạo Canonical Result và outcome ổn định cho OCR |
| Verification Database | Lưu OCR state/result và eKYC response audit trong `provider_attempts` |

Verification API, eKYC SDK endpoint, OCR Worker, Provider Adapter và Result Normalizer/Policy đều thuộc `vhm-verification-service`. eKYC SDK endpoint được publish qua dedicated streaming route của `vhm-agent-api`; control-plane API vẫn private.

OCR Worker và eKYC SDK request path phải có bulkhead/capacity guard riêng. OCR backlog không được chiếm connection, memory hoặc FPT quota dành cho luồng eKYC tương tác; ngược lại, burst eKYC không làm chậm OCR queue.

### 2.2. Lifecycle verification

`status` dưới đây chỉ mô tả vòng đời resource OCR của VHM:

```text
OCR:
QUEUED → PROCESSING → COMPLETED
```

| **Status** | **Ý nghĩa** |
| --- | --- |
| `QUEUED` | OCR verification và outbox đã commit, chờ worker claim |
| `PROCESSING` | OCR Worker đang xử lý hoặc chờ retry step |
| `COMPLETED` | Đã có outcome cuối, kể cả provider error sau hết recovery budget |
| `CANCELLED` | Bị hủy trước khi hoàn tất |
| `EXPIRED` | Quá processing deadline |

`currentStep` phục vụ OCR Worker resume. Mỗi eKYC response tạo một audit record theo `client_uuid`, operation và correlation ID.

`outcome` chỉ có khi `status=COMPLETED` và được kiểm tra theo `type`:

| **Type** | **Outcome** |
| --- | --- |
| `OCR` | `OCR_COMPLETED`, `NEED_REVIEW`, `NEED_RETRY`, `PROVIDER_ERROR` |

### 2.3. Hai execution path

Với OCR, Verification API ghi verification, media refs và outbox trong cùng transaction rồi trả `202`. Outbox Publisher đưa job đã commit vào Queue; OCR Worker claim bằng status + lease/CAS trước khi đọc media hoặc gọi FPT.

Với eKYC, Mobile tự cấu hình và khởi chạy SDK. SDK gọi init/OCR/liveness qua eKYC endpoint của `vhm-verification-service`; service stream request tới FPT, audit response rồi forward nguyên response về SDK. Không có API bootstrap/config eKYC, eKYC lifecycle API, Worker/Queue hoặc backend orchestration giữa các step.

## 3. Luồng OCR

### 3.1. Upload media

Upload diễn ra trước create OCR verification. `s3PathFile` trong response của Media Service là durable reference dùng cho `DOCUMENT_OCR`; eKYC SDK không dùng contract upload này.

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile/Web
    participant BFF as vhm-agent-api
    participant MEDIA_API as vhm-media-service
    participant STORAGE as Private Object Storage

    CLIENT->>BFF: Request Presigned PUT<br/>businessRef + purpose/role + file metadata
    BFF->>BFF: Authenticate + authorize upload path
    BFF->>MEDIA_API: Request Presigned PUT<br/>authorized context + file metadata
    MEDIA_API-->>BFF: code + message + data<br/>presignHeaders + presignedUrl + s3PathFile
    BFF-->>CLIENT: presignHeaders + presignedUrl + s3PathFile

    CLIENT->>STORAGE: PUT binary<br/>Content-Type + toàn bộ presignHeaders
    STORAGE-->>CLIENT: Upload response
```

### 3.2. Kiến trúc tổng quan

```mermaid
flowchart TB
    subgraph APP["Application layer"]
        direction LR
        CLIENT["`**Mobile/Web**
Submit OCR · Poll · Confirm`"]
        BFF["`**vhm-agent-api**
Xác thực · routing`"]
        DOMAIN["`**vhm-dossier-core**
Authorize · Apply result`"]

        CLIENT <-->|"Application API"| BFF
        BFF <-->|"OCR command/query"| DOMAIN
    end

    subgraph VERIFY["vhm-verification-service (private)"]
        direction TB
        subgraph PIPELINE[" "]
            direction LR
            API["`**Verification API**
Create · Status · Result`"]
            QUEUE[("`**Job Queue**
OCR jobs`")]
            WORKER["`**OCR Worker pool**
Internal workload`"]
            ADAPTER["`**Provider Adapter**
Session · OCR`"]
            NORMALIZE["`**Result Normalizer**
Canonical Result`"]
            API -->|"Publish committed job"| QUEUE
            QUEUE --> WORKER --> ADAPTER --> NORMALIZE
        end

        DB[("`**Verification Database**
State · Attempt · Result · Outbox`")]

        API <-->|"Persist/read"| DB
        WORKER -->|"Lease · progress"| DB
        NORMALIZE -->|"Final result"| DB
    end

    style PIPELINE fill:transparent,stroke:transparent

    STORAGE[("`**Private Object Storage**
OCR document`")]
    FPT["`**FPT AI Backend**
Session API · OCR API`"]

    DOMAIN <-->|"Private OCR command/query"| API
    WORKER -->|"HEAD/GET OCR_DOCUMENT"| STORAGE
    ADAPTER <-->|"Synchronous provider APIs"| FPT
```

### 3.3. Luồng xử lý trong `vhm-verification-service`

```mermaid
flowchart TB
    subgraph ACCEPT["A. Tiếp nhận"]
        direction LR
        API["`**Verification API**
Validate command`"]
        DB_ACCEPT[("`**Database transaction**
QUEUED · media ref · outbox`")]
        RESPONSE["`**202 Accepted**
verificationId · resourceUri`"]

        API --> DB_ACCEPT --> RESPONSE
    end

    subgraph DISPATCH["B. Phân phối job"]
        direction LR
        PUBLISHER["`**Outbox Publisher**
Publish committed job`"]
        QUEUE[("`**Job Queue**
OCR job`")]
        WORKER["`**OCR Worker pool**
Claim lease · PROCESSING`"]

        PUBLISHER --> QUEUE --> WORKER
    end

    subgraph EXECUTE["C. Xử lý document"]
        direction LR
        READ["`**Storage Reader**
HEAD/GET document`"]
        ADAPTER["`**Provider Adapter**
Session · OCR`"]
        FPT["`**FPT AI Backend**
/session/init · /ocr`"]
        NORMALIZE["`**Result Normalizer**
Canonical fields · warnings`"]
        DONE[("`**Verification Database**
COMPLETED · outcome · result`")]

        READ --> ADAPTER
        ADAPTER <-->|"Synchronous request/result"| FPT
        ADAPTER --> NORMALIZE --> DONE
    end

    DB_ACCEPT --> PUBLISHER
    WORKER --> READ
    WORKER -->|"Progress · attempt"| DONE
```

### 3.4. Sequence end-to-end

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile/Web
    participant BFF as vhm-agent-api
    participant DOMAIN as vhm-dossier-core
    participant API as Verification API
    participant QUEUE as Job Queue
    participant WORKER as OCR Worker
    participant STORAGE as Private Object Storage
    participant FPT as FPT AI Backend
    participant DB as Verification Database

    rect rgb(245, 245, 255)
    Note over CLIENT,DB: A. Submit OCR
    CLIENT->>BFF: Submit OCR<br/>dossierId + s3PathFile + documentType
    BFF->>DOMAIN: Authenticated request
    DOMAIN->>DOMAIN: Authorize dossier + OCR action<br/>validate business path prefix
    DOMAIN->>API: POST OCR<br/>businessRef + s3PathFile + documentType
    API->>DB: Commit QUEUED + media ref + outbox
    DB-->>API: Committed
    API-->>DOMAIN: 202 + verificationId + resourceUri
    DOMAIN-->>BFF: 202 + verificationId + statusUrl
    BFF-->>CLIENT: 202 + verificationId + statusUrl
    end

    rect rgb(250, 250, 235)
    Note over CLIENT,DB: B. Worker xử lý nền
    DB-->>QUEUE: Outbox Publisher<br/>OCR_JOB_CREATED
    QUEUE-->>WORKER: verificationId
    WORKER->>DB: Claim lease + PROCESSING
    WORKER->>STORAGE: HEAD/GET OCR_DOCUMENT
    STORAGE-->>WORKER: Metadata + document stream
    WORKER->>WORKER: Validate path + MIME + size
    WORKER->>FPT: POST /session/init<br/>client_uuid=verificationId
    FPT-->>WORKER: session-id
    WORKER->>FPT: POST /ocr<br/>session-id + document-type + file
    FPT-->>WORKER: OCR result
    WORKER->>WORKER: Normalize provider result
    WORKER->>DB: COMPLETED + outcome<br/>Canonical Result
    end

    rect rgb(240, 250, 245)
    Note over CLIENT,DB: C. Poll và apply kết quả
    loop Khi status chưa kết thúc
        CLIENT->>BFF: GET statusUrl
        BFF->>DOMAIN: Authorized query
        DOMAIN->>API: GET verificationId
        API->>DB: Read state/result
        DB-->>API: Current state
        API-->>DOMAIN: Status + outcome/result
        DOMAIN-->>BFF: Authorized projection
        BFF-->>CLIENT: Status + nextAction/result
    end

    CLIENT->>BFF: Confirm verificationId
    BFF->>DOMAIN: Apply confirmed OCR result
    DOMAIN->>DOMAIN: Update domain in local transaction
    end
```

### 3.5. API contract

| **Use case** | **API** | **Response** |
| --- | --- | --- |
| Tạo OCR | `POST /v1/ocr-verifications` | `202 + verificationId + resourceUri` |
| Lấy OCR | `GET /v1/ocr-verifications/{verificationId}` | Status/outcome/next action |
| Lấy OCR result | `GET /v1/ocr-verifications/{verificationId}/result` | OCR Canonical Result |
| Retry OCR | `POST /v1/ocr-verifications/{verificationId}/retries` | `202`, verification mới |

Ví dụ create OCR:

```http
POST /v1/ocr-verifications
Authorization: Bearer <service-token>
Idempotency-Key: 72aacfa4-97b8-4d0f-bb74-490f17352b1b
Content-Type: application/json

{
  "businessRef": { "type": "DOSSIER", "id": "dos-01" },
  "subjectRef": "customer-opaque-ref",
  "channel": "MOBILE",
  "platform": "ANDROID",
  "documentType": "MARRIAGE_CERTIFICATE",
  "s3PathFile": "registrations/dos-01/marriage-certificate.png"
}
```

Verification API map `s3PathFile` thành media row có role `OCR_DOCUMENT`. Sau khi verification, media refs, worker state và outbox commit, API trả:

```http
HTTP/1.1 202 Accepted
Retry-After: 3

{
  "verificationId": "ver-123",
  "type": "OCR",
  "status": "QUEUED",
  "resourceUri": "/v1/ocr-verifications/ver-123"
}
```

Ví dụ status:

```json
{
  "verificationId": "ver-123",
  "type": "OCR",
  "status": "PROCESSING",
  "currentStep": "OCR",
  "outcome": null,
  "resultAvailable": false,
  "nextAction": "POLL",
  "updatedAt": "2026-08-10T11:30:25+07:00"
}
```

Ví dụ Canonical Result:

```json
{
  "verificationId": "ver-123",
  "type": "OCR",
  "status": "COMPLETED",
  "outcome": "OCR_COMPLETED",
  "schemaVersion": "1.0",
  "document": {
    "type": "MARRIAGE_CERTIFICATE",
    "fields": {
      "certificateNumber": { "value": "01/2026", "confidence": 0.97 }
    },
    "warnings": []
  },
  "nextAction": "CONFIRM_AND_APPLY"
}
```

`nextAction` và progress được tính từ `type + status + currentStep + outcome`, không persist. OCR provider error được lưu vào attempt/outcome để Mobile/Web nhận khi poll.

## 4. Luồng eKYC

### 4.1. Phân chia trách nhiệm

Mobile khởi tạo FPT SDK với flow, `client_uuid`, Proxy Base URL của VHM và custom VHM auth header. Ở chế độ full eKYC, SDK điều khiển UI và chuỗi init session → OCR → liveness; nếu dùng OCR-only hoặc liveness-only thì Mobile chọn SDK mode tương ứng.

`client_uuid` do Mobile tạo và truyền vào SDK để correlation/audit. Khi SDK chạy, từng request đi theo cùng một cơ chế:

```text
FPT SDK
→ vhm-agent-api
→ vhm-verification-service
→ FPT AI eKYC Backend
→ vhm-verification-service audit response
→ trả nguyên response về FPT SDK
```

`vhm-verification-service` xác thực request, forward dữ liệu SDK tới FPT, audit response và trả response về SDK. Chi tiết xử lý request/response được mô tả tại mục 4.4.

Android công bố cấu hình `BASE_URL` và custom headers. iOS cần FPT cung cấp exact SDK build/config có endpoint override. Nếu SDK bắt buộc nhúng FPT API key thật trên thiết bị thì SDK build đó không phù hợp baseline này.

PDF/tài liệu nghiệp vụ không đi qua SDK eKYC mà tiếp tục dùng luồng OCR tại mục 3.

### 4.2. Kiến trúc và luồng xử lý

```mermaid
flowchart LR
    APP["`**Mobile App**
Config flow · client_uuid · Proxy Base URL · auth header`"]
    SDK["`**FPT eKYC SDK**
UI · Init session · OCR · Liveness`"]
    BFF["`**vhm-agent-api**
Auth · Streaming route`"]
    SERVICE["`**vhm-verification-service**
Forward · Audit response`"]
    DB[("`**Verification Database**
provider_attempts`")]
    FPT["`**FPT AI eKYC Backend**
Session · OCR · Liveness`"]

    APP -->|"Initialize/start"| SDK
    SDK -->|"Completion/result"| APP
    SDK <-->|"SDK request/response"| BFF
    BFF <-->|"Streaming"| SERVICE
    SERVICE <-->|"FPT wire contract"| FPT
    SERVICE -->|"Audit response"| DB
```

### 4.3. Sequence end-to-end

```mermaid
sequenceDiagram
    autonumber
    participant APP as Mobile App
    participant SDK as FPT eKYC SDK
    participant BFF as vhm-agent-api
    participant VERIFY as vhm-verification-service
    participant FPT as FPT AI eKYC Backend
    participant DB as Verification Database

    APP->>APP: Configure flow, client_uuid,<br/>Proxy Base URL + VHM auth header
    APP->>SDK: Initialize/start SDK

    loop Flow do FPT SDK tự điều phối
        SDK->>BFF: Init/OCR/Liveness request
        BFF->>VERIFY: Stream request
        VERIFY->>VERIFY: Authenticate + inject FPT credential
        VERIFY->>DB: Store request metadata<br/>không lưu media/body
        VERIFY->>FPT: Forward SDK request
        FPT-->>VERIFY: Provider response
        VERIFY->>DB: Update provider_attempts<br/>status/error + encrypted response
        VERIFY-->>BFF: Forward provider-compatible response
        BFF-->>SDK: Response
    end

    SDK-->>APP: SDK completion/result
```

### 4.4. API forwarding contract

| **Use case** | **API** | **Response** |
| --- | --- | --- |
| SDK init session | `POST /v1/ekyc-sdk/init-session` | Provider-compatible synchronous response |
| SDK OCR định danh | `POST /v1/ekyc-sdk/ocr` | Provider-compatible synchronous response |
| SDK liveness | `POST /v1/ekyc-sdk/liveness` | Provider-compatible synchronous response |

Tên/path cuối cùng phải khớp cấu hình endpoint mà FPT SDK hỗ trợ trên từng nền tảng. `vhm-verification-service` không expose generic relay và không nhận upstream URL từ request.

Ví dụ logical request; exact URL, multipart field và header phải giữ theo contract của FPT SDK:

```http
POST /v1/ekyc-sdk/ocr
Authorization: Bearer <vhm-app-token>
X-Client-UUID: <mobile-generated-client-uuid>
Content-Type: multipart/form-data; boundary=<sdk-boundary>
X-Correlation-Id: <request-id>

<FPT SDK multipart body streamed without transformation>
```

Khi nhận request từ SDK, `vhm-verification-service`:

1. Xác thực VHM token và lấy `client_uuid` phục vụ correlation/audit.
2. Kiểm tra operation/path, method, content type và input size theo contract đã allowlist.
3. Ghi metadata vào `provider_attempts`: `client_uuid`, operation, correlation ID, thời điểm bắt đầu và trạng thái gửi.
4. Stream request body tới FPT và inject provider credential/header theo contract.
5. Không lưu request body, ảnh giấy tờ hoặc selfie/video.

Khi nhận response từ FPT, `vhm-verification-service`:

1. Cập nhật `provider_attempts` với provider request ID, HTTP status, error code, thời gian hoàn tất và trạng thái xử lý.
2. Mã hóa response cần audit vào `response_payload_ciphertext` theo retention policy.
3. Trả nguyên HTTP status, header và body tương thích về SDK.
4. Không normalize eKYC response và không ghi vào `verification_results`.

Service không điều phối thứ tự init/OCR/liveness và không parse/rebuild multipart request body. Exact Android/iOS contract phải được FPT sign-off; đặc biệt iOS cần SDK build có endpoint override.

Baseline eKYC không có create/status/result API. Provider business/error response được audit và forward theo contract để SDK xử lý; không lộ credential, internal upstream hoặc stack trace.

## 5. Schema dữ liệu

### 5.1. Mô hình logic

Giữ năm bảng dùng chung; eKYC chỉ dùng `provider_attempts` để audit response, không dùng OCR lifecycle/media/result/outbox:

| **Bảng** | **Mục đích** |
| --- | --- |
| `verifications` | Business mapping, lifecycle và worker lease của OCR |
| `verification_media_refs` | Durable media ref của `DOCUMENT_OCR` |
| `provider_attempts` | OCR provider call và audit response từng eKYC SDK call |
| `verification_results` | OCR Canonical Result cuối |
| `outbox_events` | OCR job dispatch và domain event sau commit |

```mermaid
flowchart TB
    VERIFICATION["`**verifications**
OCR business mapping · lifecycle`"]
    MEDIA["`**verification_media_refs**
1:N · OCR media refs`"]
    ATTEMPT["`**provider_attempts**
OCR calls · eKYC response audit`"]
    RESULT["`**verification_results**
0:1 · final canonical result`"]
    OUTBOX["`**outbox_events**
1:N · async jobs/events`"]

    VERIFICATION --> MEDIA
    VERIFICATION -->|"OCR only"| ATTEMPT
    VERIFICATION --> RESULT
    VERIFICATION --> OUTBOX
```

eKYC rows trong `provider_attempts` được correlation bằng `client_uuid`.

### 5.2. DDL baseline

```sql
CREATE TABLE verifications (
    id                      UUID PRIMARY KEY,
    type                    VARCHAR(20) NOT NULL CHECK (type = 'OCR'),
    business_type           VARCHAR(30) NOT NULL,
    business_ref            VARCHAR(100) NOT NULL,
    subject_ref_ciphertext  BYTEA,
    consent_ref             VARCHAR(150),
    channel                 VARCHAR(20) NOT NULL CHECK (channel IN ('MOBILE', 'WEB')),
    platform                VARCHAR(20) NOT NULL CHECK (platform IN
                                ('ANDROID', 'IOS', 'WEB')),
    document_type           VARCHAR(50) NOT NULL,
    status                  VARCHAR(30) NOT NULL CHECK (status IN
                                ('QUEUED', 'PROCESSING', 'COMPLETED',
                                 'CANCELLED', 'EXPIRED')),
    current_step            VARCHAR(30) CHECK (current_step IN
                                ('VALIDATE_MEDIA', 'INIT_SESSION', 'OCR',
                                 'NORMALIZE', 'DONE')),
    outcome                 VARCHAR(30),

    attempt_count           INTEGER NOT NULL DEFAULT 0 CHECK (attempt_count >= 0),
    available_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    lease_owner             VARCHAR(100),
    lease_until             TIMESTAMPTZ,
    last_error_code         VARCHAR(80),

    retry_of                UUID REFERENCES verifications(id),
    idempotency_key         VARCHAR(100) NOT NULL,
    request_fingerprint     CHAR(64) NOT NULL,
    terminal_reason_code    VARCHAR(80),
    row_version             BIGINT NOT NULL DEFAULT 0,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at            TIMESTAMPTZ,

    CONSTRAINT uq_verification_idempotency
        UNIQUE (type, business_type, idempotency_key),
    CONSTRAINT ck_verification_channel_platform CHECK (
        (channel = 'WEB' AND platform = 'WEB') OR
        (channel = 'MOBILE' AND platform IN ('ANDROID', 'IOS'))
    ),
    CONSTRAINT ck_verification_status_outcome CHECK (
        (status <> 'COMPLETED' AND outcome IS NULL) OR
        (status = 'COMPLETED' AND outcome IN
            ('OCR_COMPLETED', 'NEED_REVIEW', 'NEED_RETRY', 'PROVIDER_ERROR'))
    ),
    CONSTRAINT ck_verification_completed CHECK (
        (status = 'COMPLETED' AND completed_at IS NOT NULL AND
            current_step = 'DONE') OR
        (status <> 'COMPLETED' AND completed_at IS NULL)
    )
);

CREATE INDEX ix_verification_business
    ON verifications (business_type, business_ref, created_at DESC);
CREATE INDEX ix_ocr_dispatch
    ON verifications (status, available_at)
    WHERE type = 'OCR' AND status IN ('QUEUED', 'PROCESSING');
CREATE INDEX ix_ocr_lease_recovery
    ON verifications (lease_until)
    WHERE type = 'OCR' AND status = 'PROCESSING';
CREATE TABLE verification_media_refs (
    id                      UUID PRIMARY KEY,
    verification_id         UUID NOT NULL REFERENCES verifications(id),
    role                     VARCHAR(30) NOT NULL CHECK (role = 'OCR_DOCUMENT'),
    position                INTEGER NOT NULL DEFAULT 1 CHECK (position > 0),
    s3_path_file             TEXT NOT NULL,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_verification_media_role_position
        UNIQUE (verification_id, role, position)
);

CREATE INDEX ix_verification_media
    ON verification_media_refs (verification_id);

CREATE TABLE provider_attempts (
    id                      UUID PRIMARY KEY,
    verification_id         UUID REFERENCES verifications(id),
    client_uuid              VARCHAR(150),
    provider                 VARCHAR(30) NOT NULL,
    operation                VARCHAR(30) NOT NULL CHECK (operation IN
                                ('INIT_SESSION', 'OCR', 'LIVENESS')),
    transport               VARCHAR(30) NOT NULL CHECK (transport IN
                                ('OCR_WORKER', 'EKYC_SDK_API')),
    attempt_no               INTEGER NOT NULL CHECK (attempt_no > 0),
    correlation_id           VARCHAR(150),
    provider_request_id      VARCHAR(150),
    provider_http_status     INTEGER,
    response_payload_ciphertext BYTEA,
    payload_key_version      VARCHAR(40),
    status                   VARCHAR(20) NOT NULL CHECK (status IN
                                ('STARTED', 'SUCCEEDED', 'FAILED', 'UNKNOWN')),
    delivery_state           VARCHAR(20) NOT NULL CHECK (delivery_state IN
                                ('NOT_SENT', 'SENDING', 'SENT', 'UNKNOWN')),
    error_class              VARCHAR(40),
    error_code               VARCHAR(80),
    started_at               TIMESTAMPTZ NOT NULL,
    finished_at              TIMESTAMPTZ,
    CONSTRAINT ck_provider_attempt_owner CHECK (
        (transport = 'OCR_WORKER' AND verification_id IS NOT NULL AND client_uuid IS NULL) OR
        (transport = 'EKYC_SDK_API' AND verification_id IS NULL AND client_uuid IS NOT NULL)
    ),
    CONSTRAINT ck_provider_attempt_response_key CHECK (
        (response_payload_ciphertext IS NULL AND payload_key_version IS NULL) OR
        (response_payload_ciphertext IS NOT NULL AND payload_key_version IS NOT NULL)
    )
);

CREATE INDEX ix_provider_attempt_verification
    ON provider_attempts (verification_id, operation, attempt_no DESC);
CREATE INDEX ix_provider_attempt_client_uuid
    ON provider_attempts (client_uuid, operation, attempt_no DESC)
    WHERE transport = 'EKYC_SDK_API';

CREATE TABLE verification_results (
    verification_id         UUID PRIMARY KEY REFERENCES verifications(id),
    version                 INTEGER NOT NULL CHECK (version > 0),
    schema_version           VARCHAR(20) NOT NULL,
    canonical_payload_ciphertext BYTEA NOT NULL,
    payload_key_version      VARCHAR(40) NOT NULL,
    is_final                 BOOLEAN NOT NULL DEFAULT FALSE,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE outbox_events (
    id                      UUID PRIMARY KEY,
    aggregate_id             UUID NOT NULL REFERENCES verifications(id),
    type                    VARCHAR(80) NOT NULL,
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

Quy ước schema:

- OCR insert một `verification_results` với `version=1`, `is_final=true` khi hoàn tất.
- Result OCR update dùng compare-and-set theo `version`; row đã `is_final=true` không được update.
- `verification_media_refs` chỉ dùng cho OCR. Baseline eKYC không persist request media; service chỉ forward stream và audit response.
- eKYC response cần audit được lưu mã hóa trong `provider_attempts`, giới hạn quyền truy cập và áp dụng retention theo chính sách dữ liệu định danh.
- Không lưu VHM token hoặc FPT credential trong database/log.
- Outbox/message chỉ chứa ID/reference tối thiểu, không chứa PII, media path, binary hoặc Canonical Result.

## 6. Tin cậy, timeout và vận hành

### 6.1. OCR worker transaction và idempotency

- Create OCR insert verification, media ref và outbox trong một transaction ngắn.
- Cùng `Idempotency-Key` và request fingerprint trả resource cũ; cùng key nhưng fingerprint khác trả `409 IDEMPOTENCY_CONFLICT`.
- Queue xử lý theo at-least-once; message chỉ chứa reference/routing tối thiểu và OCR Worker phải idempotent.
- Worker claim aggregate bằng lease/CAS; không gọi lại operation đã có provider attempt terminal thành công.
- Object Storage và provider calls luôn nằm ngoài DB transaction.
- Lỗi retry-safe đưa OCR về `QUEUED`, tăng attempt, đặt `available_at`, xóa lease và ghi outbox mới trong cùng transaction.
- Final result, `DONE + COMPLETED + outcome` commit cùng transaction.
- Outbox Publisher claim `NEW/FAILED → PUBLISHING` bằng lease; stale `PUBLISHING` được phục hồi sau lease expiry.

### 6.2. eKYC synchronous critical path

- FPT SDK gọi trực tiếp eKYC SDK endpoint của `vhm-verification-service` qua route streaming của `vhm-agent-api`; không có service trung gian riêng.
- `vhm-agent-api` và `vhm-verification-service` phải stream request end-to-end, giữ backpressure và không buffer toàn bộ document/video trong memory hoặc local disk.
- Không đặt Queue, Outbox Publisher hoặc eKYC Worker giữa SDK và FPT. Init/OCR/liveness đều là synchronous request/response.
- Service chỉ forward fixed endpoint đã allowlist. Android/iOS SDK contract, method, headers, multipart field names, response body và error behavior phải được FPT sign-off theo exact SDK version.
- Request body do SDK tạo được forward nguyên trạng. Service chỉ thêm/thay credential/header được phê duyệt; không parse rồi rebuild multipart và không sửa business response mà SDK cần đọc.
- FPT phải xác nhận SDK không yêu cầu nhúng provider API key thật trên Mobile; SDK chỉ mang VHM auth header và `vhm-verification-service` inject credential server-side.
- `client_uuid` do Mobile cấu hình và được service dùng để correlation/audit.
- Admission check phải hoàn tất trước khi đọc body. Khi DB/token service không sẵn sàng, fail fast trước khi mở upstream stream.
- Khi FPT trả response, service lưu audit data cần thiết rồi forward nguyên response tương thích về SDK.

### 6.3. Error và retry

- Provider business/error response được forward theo contract để SDK tự hiển thị hoặc retry flow.
- `vhm-verification-service` không tự retry init/OCR/liveness mutation vì SDK sở hữu session và retry behavior.
- Nếu timeout sau khi request có thể đã gửi, ghi attempt `UNKNOWN` và trả lỗi tương thích SDK; không replay body mù.
- Khi người dùng chạy lại toàn bộ eKYC flow, Mobile tạo `client_uuid` mới.

### 6.4. Timeout và cancellation

Timeout phải thỏa quan hệ:

```text
FPT outbound timeout < vhm-verification-service deadline < vhm-agent-api/SDK request timeout
```

Mỗi operation `INIT_SESSION`, `OCR`, `LIVENESS` có connect timeout, response-header timeout, streaming idle timeout và hard deadline riêng theo contract FPT. Client disconnect phải propagate cancellation xuống FPT khi transport cho phép, nhưng disconnect không chứng minh provider chưa xử lý; attempt vẫn có thể là `UNKNOWN`.

OCR outbound timeout phải ngắn hơn worker lease còn lại. OCR Worker renew lease giữa các step nếu cần; hard timeout phải hủy outbound request trước khi mất lease.

### 6.5. Quota, scaling và observability

- OCR dùng queue, worker concurrency và token bucket theo quota OCR.
- eKYC SDK route dùng admission control theo active streams, request/byte rate, tenant/flow và quota FPT; không dùng queue để giữ request tương tác.
- `vhm-verification-service` có route-level connection/concurrency limit giữa control-plane, eKYC streaming path và OCR Worker để cùng service nhưng không tranh hết tài nguyên.
- Circuit breaker OCR dừng claim job mới và dời `available_at`. Circuit breaker eKYC từ chối trước khi đọc body, trả lỗi SDK-compatible và không giữ connection treo.
- Metric/alert tối thiểu: OCR queue age/depth, outbox lag, eKYC active streams/bytes/latency/error/429/502/504, stale OCR lease và terminal outcome rate.
- Không dùng `verificationId`, subject/business ref hoặc PII làm metric label.

### 6.6. Bảo mật và lưu media eKYC

- SDK request dùng VHM auth header do Mobile cấu hình; VHM credential không được dùng thay provider credential ở upstream.
- FPT credential chỉ tồn tại trong secret manager/runtime của `vhm-verification-service`; không trả về SDK, BFF response hoặc log.
- TLS bắt buộc ở cả hai hop.
- `vhm-verification-service` enforce method/path/content type/file count/size/timeout trước hoặc trong streaming; không cung cấp generic relay và không chấp nhận upstream URL từ client.
- Không log request/response body, multipart boundary content, FPT API key, session ID, ảnh giấy tờ hoặc selfie/video.
- Baseline không persist raw eKYC media chỉ vì media đi qua service. Nếu VHM cần lưu media cho audit/manual review, phải có consent/purpose/retention và pipeline lưu riêng bằng streaming hoặc presigned upload; không buffer media trong DB transaction hay local disk của service.

## 7. Kế hoạch triển khai và kiểm thử

### 7.1. Thứ tự triển khai

1. Chốt với FPT exact Android/iOS SDK version hỗ trợ trỏ Proxy Base URL về `vhm-verification-service` và request/response wire contract.
2. Chốt hai policy độc lập: OCR `documentType → model/input limit`; eKYC SDK config trên Mobile, document/liveness mode và input limit.
3. Xây OCR schema/Verification API và eKYC response audit store.
4. Xây OCR upload/worker/provider flow như mục 3.
5. Tích hợp SDK trên Android/iOS; Mobile tự cấu hình flow, `client_uuid`, Proxy Base URL, custom VHM auth header và nhận SDK callback.
6. Xây eKYC SDK endpoint đồng bộ ngay trong `vhm-verification-service`, sau streaming route của `vhm-agent-api`.
7. Kiểm tra audit response eKYC và SDK callback success/failure.
8. Contract/load/security/resilience test và chốt timeout, active stream, bandwidth, quota, SLO/alert trước production.

### 7.2. Kiểm thử tối thiểu

| **Lớp test** | **Phạm vi** |
| --- | --- |
| Unit | OCR type/status/idempotency/canonical mapping và eKYC audit mapping |
| FPT SDK contract | Exact Android/iOS Proxy Base URL, headers, multipart field/body và response/error parsing qua `vhm-verification-service` |
| Provider contract | Init/OCR/liveness success, business error, malformed response, 429, 5xx và timeout |
| Database | CHECK/unique/index, optimistic lock, final result guard và outbox/worker lease recovery |
| Queue/Outbox | OCR duplicate/redelivery, publish-before-mark và worker restart; xác nhận SDK path không enqueue provider call |
| eKYC synchronous integration | End-to-end SDK init/OCR/liveness forwarding, streaming/backpressure, response audit, credential injection và fixed upstream allowlist |
| OCR end-to-end | Upload → create `202` → OCR Worker → result → confirm/apply |
| eKYC end-to-end | Mobile config → init SDK → SDK chạy init/OCR/liveness qua `vhm-verification-service` → audit response → SDK callback |
| Security | Token tamper/expiry/replay, UUID/session swap, cross-domain IDOR, generic-relay attempt, malicious multipart và PII-safe logs |
| Resilience | Service/provider outage, disconnect giữa upload, unknown-after-send, DB checkpoint failure, SDK resume và session expiry |
| Performance | Active streams, bandwidth, p95/p99 từng operation, service memory và OCR queue burst độc lập |

`vhm-verification-service` sở hữu OCR control/worker path. Với eKYC, Mobile tự cấu hình FPT SDK; backend chỉ forward đồng bộ, inject FPT credential và audit response, không bootstrap config, không orchestration và không có eKYC Worker.
