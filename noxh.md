# Vấn đề 6: Lựa chọn nền tảng tích hợp OCR tập trung

**User case:** Người dùng chụp hoặc upload giấy tờ trên Mobile/Web để hệ thống OCR, phân loại, gợi ý dữ liệu hoặc xác minh danh tính khi tạo hồ sơ NOXH.

**Hai journey được thiết kế độc lập:**

- **`OCR_ONLY`:** Trích xuất và chuẩn hóa dữ liệu giấy tờ. Kết quả OCR không khẳng định người thao tác là chủ thể trên giấy tờ và luôn cần người dùng kiểm tra trước khi apply vào hồ sơ.
- **`FULL_EKYC`:** OCR giấy tờ định danh kết hợp liveness và face matching để xác minh người thao tác. Journey có thể bổ sung QR/NFC theo channel và policy đã được phê duyệt.

**Loại tài liệu dự kiến:**

- CCCD mặt trước/sau.
- Giấy đăng ký kết hôn.
- Bản sao có chứng thực giấy chứng nhận hộ gia đình nghèo/cận nghèo.
- Các giấy tờ khác trong checklist hồ sơ NOXH khi có mẫu OCR tương ứng.

`vhm-verification-service` là nền tảng verification dùng chung, điều phối cả `OCR_ONLY` và `FULL_EKYC`, cô lập contract/credential của provider và chuẩn hóa kết quả. Quyết định cập nhật hồ sơ hoặc chấp nhận kết quả vẫn thuộc `vhm-dossier-core`.

**Các use case:**

1. **OCR CCCD:** Người dùng chụp đủ mặt trước/sau. Hệ thống trích xuất số định danh, họ tên, ngày sinh, giới tính, quốc tịch, quê quán, nơi thường trú, ngày cấp/hết hạn và cảnh báo chất lượng hoặc hai mặt không khớp.
2. **OCR giấy đăng ký kết hôn:** Người dùng upload đầy đủ các trang. Hệ thống trích xuất số giấy chứng nhận, thông tin hai bên, ngày đăng ký, nơi đăng ký và thông tin người ký/cơ quan cấp nếu mẫu tài liệu hỗ trợ.
3. **OCR giấy chứng nhận hộ nghèo/cận nghèo:** Người dùng upload bản sao có chứng thực. Hệ thống trích xuất số văn bản/chứng thực, chủ hộ, địa chỉ, loại xác nhận, thời gian hiệu lực, cơ quan cấp và ngày cấp.
4. **OCR giấy tờ khác:** NOXH truyền `documentType`; `vhm-verification-service` chỉ xử lý khi đã có model/template và schema kết quả tương ứng. Tài liệu chưa được hỗ trợ hoặc có confidence thấp được chuyển sang nhập liệu/kiểm tra thủ công.
5. **FULL_EKYC:** Người dùng chụp giấy tờ định danh và thực hiện liveness. Hệ thống xử lý OCR, kiểm tra liveness, so khớp khuôn mặt và trả kết quả `VERIFIED/REJECTED/NEED_RETRY` theo policy; Mobile/Web không tự quyết định kết quả từ SDK response.

OCR chỉ hỗ trợ số hóa và kiểm tra dữ liệu theo khả năng của model. Việc xác nhận bản sao có giá trị pháp lý, giấy tờ còn hiệu lực hoặc hồ sơ đáp ứng điều kiện NOXH vẫn do nghiệp vụ và người duyệt quyết định.

**Bối cảnh:** Nếu `vhm-dossier-core` tự tích hợp FPT AI, toàn bộ logic quản lý credential, provider session, async job, retry, error mapping, audit và dữ liệu định danh sẽ nằm trong service nghiệp vụ NOXH và phụ thuộc trực tiếp vào contract của provider.

| **Hướng tiếp cận** | **Ưu điểm** | **Nhược điểm** | **Lựa chọn (Yes/No)** |
| --- | --- | --- | --- |
| **`vhm-dossier-core` tích hợp trực tiếp FPT AI** | Ít thành phần; thời gian triển khai ban đầu ngắn | `vhm-dossier-core` phải xử lý credential, provider session, queue/worker, retry và mã lỗi riêng của FPT AI; thay đổi provider tác động trực tiếp nghiệp vụ NOXH; tăng rủi ro dữ liệu định danh trong service hồ sơ | **No** |
| **`vhm-verification-service`** (vai trò Verification Provider Proxy) | `vhm-dossier-core` chỉ authorize media đã finalize, tạo yêu cầu xử lý và tra cứu Canonical Result; toàn bộ async job, provider integration, credential, session, retry, quota, lưu trữ kết quả, bảo mật, audit và chuẩn hóa được xử lý tại service; thay provider không yêu cầu sửa nghiệp vụ NOXH | Thêm một service và một network hop; `vhm-verification-service` phải đáp ứng HA/SLA vì là dependency của NOXH | **Yes** |

**Phương án chọn:** Sử dụng **`vhm-verification-service`** làm lớp tích hợp tập trung cho cả OCR và eKYC, với vai trò kiến trúc là Verification Provider Proxy. `vhm-dossier-core` kiểm tra quyền nghiệp vụ, quản lý attachment, gửi yêu cầu xử lý media đã finalize và cho phép tra cứu Canonical Result; không gọi trực tiếp hoặc phụ thuộc contract của FPT AI.

**Nguyên tắc kiến trúc:**

- Giữ chung một service nhưng tách handler và state transition của `OCR_ONLY`/`FULL_EKYC`; không tách thành hai service OCR và eKYC.
- `vhm-verification-service` là private internal service. Mobile/Web chỉ đi qua `vhm-agent-api` và `vhm-dossier-core`; provider callback nếu có phải qua Public Callback Ingress/API Gateway.
- Provider credential chỉ tồn tại tại Provider Adapter. Không truyền credential, provider session hoặc raw provider payload về Mobile/Web, BFF hoặc `vhm-dossier-core`.
- `status` mô tả vòng đời kỹ thuật; `outcome` mô tả kết luận OCR/eKYC. Lỗi kỹ thuật không được ánh xạ thành `REJECTED`.
- Mọi create/retry/submit phải idempotent; job chỉ được trả thành công sau khi đã persist và bảo đảm enqueue bằng transactional outbox hoặc cơ chế tương đương.

**Các module chính của `vhm-verification-service`:**

| **Module** | **Trách nhiệm implementation** |
| --- | --- |
| Verification API | Nhận internal command/query từ `vhm-dossier-core`; validate contract, idempotency key và authorized media reference; trả `verificationId/status/result` |
| Journey Orchestrator | Chọn `OCR_ONLY` hoặc `FULL_EKYC`, kiểm tra milestone bắt buộc và điều khiển state transition |
| OCR Worker | Đọc media đã finalize, xử lý ảnh định danh hoặc tách trang/batch cho tài liệu lớn, theo dõi progress và tổng hợp kết quả |
| eKYC Worker | Điều phối provider session, OCR front/back, liveness, face matching và QR/NFC tùy policy |
| Provider Adapter | Inject server credential; ánh xạ request/response/error; quản lý timeout, quota, circuit breaker và provider session |
| Result Normalizer | Chuẩn hóa field, confidence, warning, liveness/face-match check thành Canonical Result có version |
| Decision Mapper | Ánh xạ canonical checks thành journey outcome; không chứa rule phê duyệt hồ sơ NOXH |
| Persistence + Outbox | Lưu session/job/check/result/history; publish job/event an toàn; optimistic locking và dedupe |
| Reconciliation Worker | Khôi phục provider job/result treo bằng bounded polling hoặc Result API; không polling liên tục mọi session |
| Security/Audit/Observability | Authorization defense-in-depth, masking, audit, metrics, trace và cảnh báo backlog/provider |

**Khả năng FPT AI đã đối chiếu:**

- FPT hỗ trợ ba phương thức tích hợp Mobile SDK, Web SDK và backend API; API linh hoạt nhất nhưng phía khách hàng phải tự xây UI và luồng xử lý ([So sánh các phương thức tích hợp](https://docs-vision.fpt.ai/ekyc/III-integration/III-0-so-sanh/)).
- FPT eKYC backend API yêu cầu khởi tạo session trước khi gọi `/ocr`; OCR hỗ trợ ba nhóm giấy tờ `idr`, `passport`, `dlr` và trả kết quả trong response của request ([Các API của luồng cập nhật thông tin](https://docs-vision.fpt.ai/ekyc/III-integration/III-2-APIs/a-APIs%20of%20eKYC%20Flows/APIs-in-update-information-flow/)). Tài liệu public chưa mô tả API `submit job/get job status` riêng cho OCR bất đồng bộ, do đó VHM tự quản lý async job và worker ở lớp `vhm-verification-service`.
- Callback và `POST /callback/get_result` là các cách lấy/đối soát dữ liệu phiên eKYC, không phải API để Mobile/Web theo dõi verification job nội bộ của VHM ([Lấy dữ liệu từ hệ thống](https://docs-vision.fpt.ai/ekyc/III-integration/III-2-APIs/a-APIs%20of%20eKYC%20Flows/APIs-result/)).
- Web SDK có `ocrFiles` trong `on_result`, tức là sau khi đã có kết quả; tài liệu Mobile SDK public chưa xác nhận callback trả raw file ngay sau capture. Vì vậy flow upload Presigned URL không phụ thuộc capability chưa được xác nhận của FPT SDK ([Web SDK](https://docs-vision.fpt.ai/ekyc/III-integration/III-1-SDKs/web-sdk/)).
- Giấy đăng ký kết hôn, giấy chứng nhận hộ nghèo/cận nghèo và PDF nhiều trang không thuộc ba loại giấy tờ của FPT eKYC `/ocr`; cần model/template của FPT AI Read hoặc Document OCR Provider khác và phải xác nhận riêng input limit, số trang, SLA, cơ chế synchronous/async trước khi triển khai.

**Luồng 1 — Upload tài liệu:**

```mermaid
flowchart LR
    AGENT["`**Mobile/Web**
Capture · Upload`"]
    BFF["`**vhm-agent-api**
Xác thực · routing`"]
    NOXH["`**vhm-dossier-core**
Authorize · Presigned URL · Media metadata`"]
    MEDIA[("`**Private Object Storage**
Dossier attachments`")]

    AGENT -->|"1. Request upload slot"| BFF
    BFF -->|"2. Authorize + file metadata"| NOXH
    NOXH -->|"3. mediaId + Presigned URL"| BFF
    BFF -->|"4. mediaId + Presigned URL"| AGENT
    AGENT ==>|"`5. Presigned PUT
binary + checksum`"| MEDIA
    AGENT -->|"6. Submit mediaId + object version"| BFF
    BFF -->|"7. Finalize media"| NOXH
    NOXH <-->|"8. HEAD/validate object"| MEDIA
```

Upload là luồng độc lập với OCR. `vhm-dossier-core` chưa tạo `verificationId` và `vhm-verification-service` không tham gia cho đến khi attachment đã được finalize và người dùng yêu cầu OCR.

**Luồng 2 — Kiến trúc xử lý OCR bất đồng bộ:**

```mermaid
flowchart LR
    CLIENT["`**Mobile/Web**
Submit OCR · Poll result`"]
    BFF["`**vhm-agent-api**
Xác thực · routing`"]
    DOSSIER["`**vhm-dossier-core**
Authorize · Apply result`"]
    VERIFY["`**vhm-verification-service**
Async OCR · eKYC · Provider Adapter`"]
    MEDIA[("`**Private Object Storage**
Finalized attachments`")]
    PROVIDER["`**OCR/eKYC Provider**
FPT eKYC · Document OCR`"]
    DB[("`**Verification Database**
Job status · Canonical result`")]

    CLIENT -->|"Submit OCR · GET status"| BFF
    BFF -->|"Authenticated request"| DOSSIER
    DOSSIER -->|"Create job · Query status"| VERIFY

    VERIFY -->|"Worker: GET media"| MEDIA
    VERIFY ==>|"Worker: OCR/eKYC request"| PROVIDER
    PROVIDER ==>|"Provider result"| VERIFY
    VERIFY <-->|"Persist/read job"| DB

    VERIFY -.->|"202 QUEUED · Status · Result"| DOSSIER
    DOSSIER -.->|"Authorized response"| BFF
    BFF -.->|"Response"| CLIENT
```

Đường liền thể hiện request/xử lý; đường nét đứt thể hiện response `202`, trạng thái và kết quả. Chart này chỉ mô tả thành phần và hướng dữ liệu; thứ tự xử lý chi tiết được thể hiện tại sequence diagram trong phần **Cách hoạt động**.

Request tạo OCR chỉ chờ đến khi `vhm-verification-service` persist job và trả `verificationId + QUEUED`; việc đọc object và gọi provider diễn ra trong worker. Mobile/Web không giữ kết nối chờ provider mà tra cứu trạng thái/kết quả qua `vhm-agent-api` và `vhm-dossier-core`.

Giải pháp này tách nghiệp vụ NOXH khỏi chi tiết tích hợp provider. Chi phí của một service và một network hop được chấp nhận để async job, credential, quota, provider result, audit và vận hành provider được quản lý tập trung tại `vhm-verification-service`; attachment upload vẫn thuộc `vhm-dossier-core`.

**Phân định ownership:**

| **Năng lực** | **`vhm-dossier-core`** | **`vhm-verification-service`** |
| --- | --- | --- |
| Nghiệp vụ | Kiểm tra quyền trên hồ sơ/dự án; gửi `businessRef`; quyết định sử dụng kết quả | Không xử lý rule nghiệp vụ NOXH |
| Upload tài liệu | Sở hữu `mediaId`, object key, Presigned URL, attachment metadata, finalize và retention; binary nằm trong Private Object Storage | Không cấp Presigned URL và không sở hữu attachment; chỉ đọc object đã finalize khi có yêu cầu OCR được ủy quyền |
| API tích hợp | Sau khi authorize media đã finalize, gọi API nội bộ `create/status/result` với `businessRef`, `documentType`, authorized `mediaRef` và `idempotencyKey` | Quản lý async job, provider API contract, model/template và provider credential |
| Phiên xử lý | Không quản lý state machine OCR; authorize mọi request tạo job và tra cứu | Sinh `verificationId` khi nhận yêu cầu OCR; persist `QUEUED` trước khi trả `202`; quản lý queue/worker, provider session, attempt, timeout, idempotency, progress và trạng thái |
| Kết quả | Proxy API tra cứu trạng thái/kết quả sau khi kiểm tra quyền; chỉ apply kết quả đã được người dùng xác nhận | Nhận synchronous result trong worker hoặc theo dõi provider async nếu provider hỗ trợ; chuẩn hóa field, confidence, warning và error code thành Canonical Result |
| Dữ liệu | Database lưu attachment metadata/reference, checksum, object version và retention; không lưu binary hoặc raw provider payload | Database chỉ lưu verification session, Canonical Result và provider metadata tối thiểu; không lưu binary hoặc sở hữu attachment reference lâu dài |
| Resilience | Không retry trực tiếp với provider | Durable queue, retry có giới hạn, reconciliation, circuit breaker, rate limit, quota và xử lý từng trang/batch với tài liệu lớn |
| Audit/Monitoring | Chỉ audit việc áp dụng kết quả vào nghiệp vụ | Audit toàn bộ vòng đời OCR và vận hành provider |

Nhờ đó, `vhm-dossier-core` không phải triển khai provider adapter, queue/worker, provider session, database verification, retry, security và monitoring cho provider.

**Trách nhiệm của `vhm-verification-service`:**

- Khi nhận yêu cầu OCR cho media đã finalize, sinh `verificationId`, liên kết với `businessRef`, persist trạng thái `QUEUED`, enqueue job rồi trả kết quả tạo job; không tạo verification session trong luồng upload.
- Chỉ nhận `mediaId/mediaRef` đã được `vhm-dossier-core` authorize và finalize; không cấp Presigned URL, không nhận object path từ Mobile/Web và không quản lý attachment nghiệp vụ.
- Đọc object từ Private Object Storage bằng quyền read-only giới hạn theo exact object; kiểm tra lại binding, MIME/magic bytes và kích thước trước khi gửi provider.
- Nhận `documentType`, chọn model/template OCR tương ứng và chuẩn hóa kết quả theo schema của từng loại tài liệu.
- Với CCCD/GPLX/Hộ chiếu, worker khởi tạo FPT eKYC session bằng `client_uuid=verificationId`, `only-engine=1`, sau đó gọi `/ocr` và ghi nhận synchronous provider result.
- Với PDF/tài liệu nhiều trang, worker stream file, kiểm tra giới hạn, tách trang/batch, xử lý với concurrency giới hạn và retry theo từng trang. Khi kết thúc, `status=COMPLETED` và `outcome` là `OCR_COMPLETED`, `PARTIAL`, `NEED_REVIEW` hoặc `PROVIDER_ERROR`; chỉ sử dụng provider/model đã được xác nhận hỗ trợ loại tài liệu tương ứng.
- Nếu một provider thực sự cung cấp async job, Provider Adapter lưu `providerJobId` và tự poll/reconcile; Mobile/Web không gọi provider status API. Callback từ provider chỉ được bổ sung qua Public Callback Ingress/API Gateway, không gọi thẳng vào `vhm-verification-service` private.
- Chuẩn hóa payload và error code của FPT AI thành contract nội bộ ổn định, không trả raw provider payload cho NOXH.
- Quản lý retry, timeout, circuit breaker, quota/rate limit và cô lập sự cố provider khỏi `vhm-dossier-core`.
- Áp dụng kiểm soát truy cập, mã hóa dữ liệu nhạy cảm, masking, audit và chính sách lưu/xóa verification session, Canonical Result và provider payload tối thiểu.
- Cung cấp API để NOXH tạo async job, tra cứu trạng thái/progress và lấy Canonical Result.

**Trách nhiệm tối thiểu của `vhm-dossier-core`:**

- Kiểm tra người dùng có quyền thao tác hồ sơ/dự án.
- Sinh `mediaId` và exact object key, cấp Presigned PUT URL sau authorization, lưu attachment metadata và finalize object sau khi HEAD/validate checksum/version thành công.
- Quản lý orphan cleanup, retention và quyền truy cập attachment theo hồ sơ/dự án.
- Chỉ sau khi media đã `FINALIZED`, gửi `businessRef=dossierId`, `documentType`, authorized `mediaRef` và `idempotencyKey` để tạo verification job; không truyền object path do client cung cấp.
- Kiểm tra quyền trên mỗi request tra cứu, gọi API `status/result` của `vhm-verification-service` và bind Canonical Result với đúng hồ sơ/media trước khi trả Mobile/Web.
- Cho người dùng xác nhận dữ liệu trước khi cập nhật khách hàng hoặc hồ sơ NOXH.

## Luồng chi tiết `OCR_ONLY`

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile/Web
    participant BFF as vhm-agent-api
    participant NOXH as vhm-dossier-core
    participant OCR as vhm-verification-service
    participant QUEUE as Verification Job Queue
    participant MEDIA as Private Object Storage
    participant PROVIDER as OCR/eKYC Provider

    CLIENT->>CLIENT: Chụp/chọn file<br/>kiểm tra sơ bộ + tính checksum
    CLIENT->>BFF: Xin presigned upload URL<br/>dossierId + documentType + file metadata
    BFF->>NOXH: Authorize và create media slot idempotent
    NOXH->>NOXH: Cấp mediaId + exact object key + expiry<br/>sinh Presigned PUT URL
    NOXH-->>BFF: Presigned PUT URL + required headers
    BFF-->>CLIENT: Presigned PUT URL + mediaId

    CLIENT->>MEDIA: PUT binary trực tiếp<br/>content-length + content-type + checksum
    MEDIA-->>CLIENT: ETag + object version
    CLIENT->>BFF: Submit mediaId + checksum + object version
    BFF->>NOXH: Authorized media submit
    NOXH->>MEDIA: HEAD object và đọc metadata
    MEDIA-->>NOXH: size + content-type + checksum + version
    NOXH->>NOXH: Validate checksum và finalize attachment

    CLIENT->>BFF: Yêu cầu OCR<br/>dossierId + mediaId + documentType
    BFF->>NOXH: Xác thực và routing
    NOXH->>NOXH: Authorize hồ sơ + media FINALIZED
    NOXH->>OCR: Create verification job<br/>businessRef + documentType + authorized mediaRef + idempotencyKey
    OCR->>OCR: Sinh verificationId<br/>persist QUEUED
    OCR->>QUEUE: Enqueue verificationId
    OCR-->>NOXH: verificationId + QUEUED
    NOXH-->>BFF: 202 + verificationId + statusUrl
    BFF-->>CLIENT: 202 + verificationId + statusUrl

    QUEUE-->>OCR: Worker nhận job
    OCR->>OCR: Chuyển PROCESSING
    OCR->>MEDIA: GET object bằng private credential
    MEDIA-->>OCR: Object stream
    OCR->>OCR: Validate binding/MIME/magic bytes/size

    alt CCCD/GPLX/Hộ chiếu
        OCR->>PROVIDER: POST /session/init<br/>client_uuid=verificationId + only-engine=1
        PROVIDER-->>OCR: provider session-id
        OCR->>PROVIDER: POST /ocr multipart<br/>server credential + document-type
        PROVIDER-->>OCR: Synchronous OCR Result
    else PDF/tài liệu nhiều trang
        OCR->>OCR: Tách trang/batch<br/>giới hạn concurrency
        loop Từng trang/batch
            OCR->>PROVIDER: Document OCR request
            PROVIDER-->>OCR: Page/batch result
        end
        OCR->>OCR: Tổng hợp kết quả các trang<br/>outcome=OCR_COMPLETED/PARTIAL/PROVIDER_ERROR
    end

    OCR->>OCR: Chuẩn hóa Canonical Result
    OCR->>OCR: Persist status=COMPLETED<br/>Canonical Result + outcome

    loop Cho đến khi có trạng thái kết thúc
        CLIENT->>BFF: GET statusUrl
        BFF->>NOXH: Authorized status query
        NOXH->>OCR: GET status/result(verificationId)
        OCR-->>NOXH: Status + outcome/progress/result
        NOXH-->>BFF: Status/progress/Canonical Result
        BFF-->>CLIENT: Status/progress/Canonical Result
    end

    CLIENT->>BFF: Xác nhận sử dụng kết quả
    BFF->>NOXH: Apply confirmed result
    NOXH->>NOXH: Bind và cập nhật hồ sơ
```

## Luồng chi tiết `FULL_EKYC`

```mermaid
sequenceDiagram
    autonumber
    participant CLIENT as Mobile/Web
    participant BFF as vhm-agent-api
    participant NOXH as vhm-dossier-core
    participant VERIFY as vhm-verification-service
    participant QUEUE as Verification Job Queue
    participant MEDIA as Private Object Storage
    participant PROVIDER as FPT eKYC Backend

    CLIENT->>BFF: Start FULL_EKYC<br/>dossierId + subjectRef + consentRef + channel
    BFF->>NOXH: Xác thực và routing
    NOXH->>NOXH: Authorize hồ sơ/subject/purpose
    NOXH->>VERIFY: Create FULL_EKYC session<br/>businessRef + subjectRef + consentRef + channel
    VERIFY->>VERIFY: Sinh verificationId + attempt=1<br/>status=WAITING_MEDIA
    VERIFY-->>NOXH: verificationId + capture policy
    NOXH-->>BFF: eKYC bootstrap không chứa provider credential
    BFF-->>CLIENT: verificationId + capture policy

    CLIENT->>CLIENT: Capture front/back<br/>selfie hoặc liveness video
    CLIENT->>BFF: Xin upload slots<br/>verificationId + media metadata/checksum
    BFF->>NOXH: Authorize và create media slots
    NOXH-->>BFF: mediaIds + Presigned URLs
    BFF-->>CLIENT: mediaIds + Presigned URLs
    CLIENT->>MEDIA: PUT media trực tiếp<br/>document + face/liveness artifacts
    MEDIA-->>CLIENT: ETag + object version
    CLIENT->>BFF: Submit media manifest<br/>verificationId + mediaIds + checksums
    BFF->>NOXH: Authorized manifest submit
    NOXH->>MEDIA: HEAD/validate objects
    MEDIA-->>NOXH: Object metadata
    NOXH->>NOXH: Finalize media và bind verificationId/attempt
    NOXH->>VERIFY: Submit authorized mediaRefs
    VERIFY->>VERIFY: Validate required media<br/>status=QUEUED
    VERIFY->>QUEUE: Enqueue verificationId
    VERIFY-->>NOXH: 202 + QUEUED + statusUrl
    NOXH-->>BFF: 202 + statusUrl
    BFF-->>CLIENT: 202 + statusUrl

    QUEUE-->>VERIFY: eKYC Worker nhận job
    VERIFY->>VERIFY: status=PROCESSING
    VERIFY->>MEDIA: GET exact finalized objects
    MEDIA-->>VERIFY: Object streams
    VERIFY->>PROVIDER: POST /session/init<br/>client_uuid=verificationId
    PROVIDER-->>VERIFY: provider session-id
    VERIFY->>PROVIDER: POST /ocr<br/>front/back + document-type
    PROVIDER-->>VERIFY: OCR result
    VERIFY->>PROVIDER: POST /face/liveness<br/>selfie hoặc video
    PROVIDER-->>VERIFY: Liveness + face-match result

    opt Journey yêu cầu QR hoặc NFC
        VERIFY->>PROVIDER: POST /qrcode hoặc /check_chip
        PROVIDER-->>VERIFY: QR/NFC verification result
    end

    VERIFY->>VERIFY: Normalize checks + Decision Mapper
    VERIFY->>VERIFY: status=COMPLETED<br/>outcome=VERIFIED/REJECTED/NEED_RETRY/PROVIDER_ERROR

    loop Poll đến khi COMPLETED/CANCELLED/EXPIRED
        CLIENT->>BFF: GET statusUrl
        BFF->>NOXH: Authorized status query
        NOXH->>VERIFY: GET status/result
        VERIFY-->>NOXH: Status + Canonical Result/nextAction
        NOXH-->>BFF: Authorized result projection
        BFF-->>CLIENT: Status/outcome/nextAction
    end
```

`FULL_EKYC` theo kiến trúc trên yêu cầu capture component cung cấp được raw document/selfie/liveness artifact để client upload vào VHM Object Storage trước khi server-side worker gọi provider. Nếu phiên bản FPT SDK chỉ tự gửi media tới FPT và không expose artifact trước khi xử lý, phải chốt một ADR khác cho SDK proxy data-plane; không được tự động cho SDK gọi thẳng FPT hoặc mở public `vhm-verification-service`.

**Luồng capture/upload tài liệu:**

- Mobile/Web chụp hoặc chọn file bằng capture/upload component của ứng dụng. Cả ảnh chụp và file có sẵn đều dùng chung cơ chế Presigned URL đã chọn tại Vấn đề 3.
- Mobile/Web gọi `vhm-agent-api` để xin upload URL. `vhm-agent-api` chỉ xác thực/routing; `vhm-dossier-core` authorize hồ sơ, sinh `mediaId`/exact object key và trả Presigned URL. `vhm-verification-service` không tham gia luồng cấp URL.
- Presigned URL phải có TTL ngắn và bind exact object key, method, content type, content length, checksum và `mediaId`. Upload chưa có `verificationId`; client không có quyền list/read object hoặc tự chọn object key.
- Sau upload, Mobile/Web submit `mediaId`, checksum và object version qua `vhm-agent-api`. `vhm-dossier-core` kiểm tra lại quyền, HEAD/validate object rồi chuyển attachment sang `FINALIZED`. Việc finalize không tự động OCR; chỉ request OCR riêng của người dùng mới tạo verification job.
- Mobile/Web không gọi thẳng provider hoặc `vhm-verification-service`, không nhận provider API key và không gửi binary qua `vhm-agent-api`.
- Nếu sử dụng FPT SDK làm capture component, phải xác nhận phiên bản SDK cung cấp raw file/Blob trước khi SDK tự gọi FPT. Khi capability này chưa được xác nhận, sử dụng camera/file picker của ứng dụng cho flow Presigned URL.
- Provider `api-key`, `session-id`, `device-type` và `document-type` chỉ được `vhm-verification-service` inject ở outbound request. Mobile/Web không nhìn thấy provider credential.
- Binary upload nằm trong Private Object Storage do `vhm-dossier-core` quản lý. Dossier database chỉ lưu protected object reference, checksum, object version, media metadata và thời hạn xóa; upload chưa finalize được orphan-cleanup theo TTL, object đã finalize được purge theo retention policy.

**Xử lý OCR theo loại tài liệu:**

- **Ảnh định danh:** FPT eKYC `/ocr` hỗ trợ `idr`, `passport` và `dlr`. Worker khởi tạo provider session sau khi nhận job, lấy ảnh đã finalize từ Object Storage và gọi provider synchronous; client vẫn sử dụng flow async `202 + polling` của VHM.
- **PDF/tài liệu nhiều trang:** Không gửi trực tiếp vào FPT eKYC `/ocr`. Worker stream và tách trang/batch, sau đó chọn FPT AI Read hoặc Document OCR Provider/model đã được xác nhận hỗ trợ loại tài liệu. Retry và progress được quản lý theo từng trang/batch; kết quả có thể là `PARTIAL`.
- `vhm-verification-service` không tin object path từ client. Service chỉ đọc exact object do `vhm-dossier-core` ủy quyền, kiểm tra binding, MIME/magic bytes, kích thước, thứ tự mặt/trang và attempt trước khi gửi provider.
- Không retry mù request sau khi body đã được gửi. Retry chỉ thực hiện theo idempotency contract của provider hoặc tạo attempt/page job mới có kiểm soát để tránh tính phí và kết quả trùng.

## Thiết kế API

`vhm-verification-service` chỉ mở API private dưới `/internal/v1`. Kết nối từ `vhm-dossier-core` dùng mTLS và service token có scope; `Idempotency-Key` bắt buộc với mọi command tạo mới, submit hoặc retry. Mobile/Web chỉ nhìn thấy API của `vhm-dossier-core`; `vhm-agent-api` làm nhiệm vụ xác thực và routing.

| **Use case** | **API của `vhm-dossier-core` cho Mobile/Web** | **API private của `vhm-verification-service`** | **Kết quả** |
| --- | --- | --- | --- |
| Tạo OCR | `POST /dossiers/{dossierId}/ocr-verifications` | `POST /internal/v1/verifications/ocr` | `202`, tạo job ở `QUEUED` |
| Bắt đầu eKYC | `POST /dossiers/{dossierId}/ekyc-verifications` | `POST /internal/v1/verifications/ekyc` | `201`, tạo session ở `WAITING_MEDIA` và trả capture policy |
| Submit media eKYC | `POST /dossiers/{dossierId}/verifications/{id}/media` | `POST /internal/v1/verifications/{id}/media-submissions` | `202`, validate manifest rồi chuyển `QUEUED` |
| Lấy trạng thái | `GET /dossiers/{dossierId}/verifications/{id}` | `GET /internal/v1/verifications/{id}` | Status, progress, outcome, next action |
| Lấy kết quả | `GET /dossiers/{dossierId}/verifications/{id}/result` | `GET /internal/v1/verifications/{id}/result` | Canonical Result đã lọc/mask |
| Hủy | `POST /dossiers/{dossierId}/verifications/{id}/cancel` | `POST /internal/v1/verifications/{id}/cancel` | Hủy ngay khi chưa chạy hoặc best-effort khi đang xử lý |
| Thử lại | `POST /dossiers/{dossierId}/verifications/{id}/retries` | `POST /internal/v1/verifications/{id}/retries` | Tạo attempt mới, không reset hoặc ghi đè attempt cũ |

`vhm-verification-service` trả `resourceUri` nội bộ. `vhm-dossier-core` ánh xạ URI này thành `statusUrl` thuộc dossier và authorize lại mỗi lần Mobile/Web poll; không chuyển nguyên internal URI cho client.

Request tạo `OCR_ONLY` sau khi attachment đã `FINALIZED`:

```http
POST /internal/v1/verifications/ocr
Authorization: Bearer <service-token>
Idempotency-Key: 72aacfa4-97b8-4d0f-bb74-490f17352b1b
Content-Type: application/json

{
  "businessRef": { "type": "DOSSIER", "id": "dos-01" },
  "subjectRef": "customer-opaque-ref",
  "channel": "MOBILE",
  "documentType": "IDR",
  "media": [
    {
      "mediaId": "media-front-01",
      "role": "DOCUMENT_FRONT",
      "authorizedMediaRef": "opaque-signed-media-ref",
      "checksumSha256": "...",
      "objectVersion": "..."
    }
  ]
}
```

```http
HTTP/1.1 202 Accepted
Retry-After: 3

{
  "verificationId": "ver-123",
  "journey": "OCR_ONLY",
  "status": "QUEUED",
  "resourceUri": "/internal/v1/verifications/ver-123"
}
```

Request bắt đầu và submit `FULL_EKYC` được tách làm hai bước:

```http
POST /internal/v1/verifications/ekyc
Idempotency-Key: 434f4412-9027-4fb4-8ad3-05d98f8e68e0

{
  "businessRef": { "type": "DOSSIER", "id": "dos-01" },
  "subjectRef": "customer-opaque-ref",
  "consentRef": "consent-opaque-ref",
  "channel": "MOBILE",
  "policyKey": "NOXH_IDENTITY_V1"
}
```

Response `201 Created` trả `verificationId`, `status=WAITING_MEDIA`, `attempt=1`, `expiresAt` và `capturePolicy.requiredMedia`; tuyệt đối không chứa FPT API key hoặc provider session.

```http
POST /internal/v1/verifications/ver-456/media-submissions
Idempotency-Key: 88367e76-e93d-40fb-82c9-619a11fb41a8

{
  "attempt": 1,
  "media": [
    { "mediaId": "m-front", "role": "DOCUMENT_FRONT", "authorizedMediaRef": "...", "checksumSha256": "...", "objectVersion": "..." },
    { "mediaId": "m-back", "role": "DOCUMENT_BACK", "authorizedMediaRef": "...", "checksumSha256": "...", "objectVersion": "..." },
    { "mediaId": "m-live", "role": "LIVENESS", "authorizedMediaRef": "...", "checksumSha256": "...", "objectVersion": "..." }
  ]
}
```

`authorizedMediaRef` là reference opaque do backend phát hành, bind với `mediaId`, exact object/version, checksum, business reference và caller. API không nhận bucket/key, URL hoặc path do client tự truyền. Với job bị trì hoãn quá thời hạn media grant, worker yêu cầu cấp lại grant nội bộ; không nới quyền đọc toàn bucket.

Status response dùng chung cho hai journey:

```json
{
  "verificationId": "ver-123",
  "journey": "OCR_ONLY",
  "status": "PROCESSING",
  "outcome": null,
  "progress": {
    "totalUnits": 20,
    "processedUnits": 8,
    "failedUnits": 0
  },
  "resultAvailable": false,
  "nextAction": "POLL",
  "updatedAt": "2026-08-10T11:30:25+07:00"
}
```

Quy ước HTTP/error:

| **HTTP** | **Sử dụng** |
| --- | --- |
| `200/201/202` | Query thành công; tạo resource; command đã được persist và chấp nhận xử lý |
| `400` | Contract/header sai định dạng |
| `401/403` | Service identity hoặc scope không hợp lệ |
| `404` | Không tồn tại hoặc caller không được phép biết resource tồn tại |
| `409` | Idempotency key trùng nhưng fingerprint khác; state transition sai; result chưa sẵn sàng |
| `413/415/422` | Media vượt giới hạn, sai content type/magic bytes hoặc thiếu/sai logical part |
| `429` | Admission control/quota nội bộ; trả `Retry-After` |
| `503` | Không thể persist/enqueue an toàn; không trả `202` giả |

Error body chỉ dùng canonical `code`, `message`, `retryable`, `correlationId`; không trả raw FPT code/message, stack trace, media reference hoặc credential. Lỗi provider xảy ra trong worker được thể hiện bằng job retry và final `outcome`, không biến thành HTTP error của request polling.

## Lifecycle và outcome

`status` chỉ thể hiện vòng đời kỹ thuật. `outcome` chỉ được gán khi `status=COMPLETED`; vì vậy một lần xác minh bị lỗi provider sau khi hết recovery budget vẫn là `COMPLETED + PROVIDER_ERROR`, không phải `REJECTED`.

```mermaid
stateDiagram-v2
    [*] --> WAITING_MEDIA: Create FULL_EKYC
    [*] --> QUEUED: Create OCR_ONLY
    WAITING_MEDIA --> QUEUED: Submit đủ finalized media
    WAITING_MEDIA --> CANCELLED: User hủy
    WAITING_MEDIA --> EXPIRED: Hết capture TTL
    QUEUED --> PROCESSING: Worker claim job
    QUEUED --> CANCELLED: Hủy trước khi claim
    QUEUED --> EXPIRED: Quá processing deadline
    PROCESSING --> COMPLETED: Persist Canonical Result
    PROCESSING --> CANCEL_REQUESTED: Hủy trong khi gọi provider
    CANCEL_REQUESTED --> CANCELLED: Worker dừng an toàn
    COMPLETED --> [*]
    CANCELLED --> [*]
    EXPIRED --> [*]
```

| **Journey** | **Outcome khi `COMPLETED`** | **Ý nghĩa** |
| --- | --- | --- |
| `OCR_ONLY` | `OCR_COMPLETED` | Đọc đủ tài liệu; vẫn cần người dùng xác nhận dữ liệu |
| `OCR_ONLY` | `PARTIAL` | Chỉ một phần trang/field xử lý thành công |
| `OCR_ONLY` | `NEED_REVIEW` | Confidence/warning vượt ngưỡng cần kiểm tra thủ công |
| Cả hai | `NEED_RETRY` | Media hoặc thao tác người dùng có thể thực hiện lại |
| `FULL_EKYC` | `VERIFIED` | Document, liveness và face match đạt policy |
| `FULL_EKYC` | `REJECTED` | Check nghiệp vụ xác minh không đạt policy; không dùng cho lỗi transport/provider |
| Cả hai | `PROVIDER_ERROR` | Hết retry/reconciliation budget hoặc provider không trả kết quả hợp lệ |

Worker có trạng thái riêng `PENDING/RUNNING/RETRY_WAIT/SUCCEEDED/DEAD`. `RETRY_WAIT` và `DEAD` là chi tiết vận hành của job, không được trả trực tiếp thành lifecycle/outcome cho Mobile/Web.

## Canonical Result

Canonical Result có `schemaVersion`, không phụ thuộc tên field hoặc error code của FPT. API Result chỉ trả field nằm trong allowlist theo purpose; PII được mask theo role và raw score chỉ dành cho rule engine/audit có quyền.

```json
{
  "verificationId": "ver-123",
  "journey": "FULL_EKYC",
  "status": "COMPLETED",
  "outcome": "VERIFIED",
  "schemaVersion": "1.0",
  "document": {
    "type": "IDR",
    "fields": {
      "identityNumber": { "value": "0012******89", "confidence": 0.99 },
      "fullName": { "value": "NGUYEN VAN A", "confidence": 0.98 }
    },
    "warnings": []
  },
  "checks": [
    { "type": "DOCUMENT_OCR", "result": "PASS", "reasonCode": null },
    { "type": "LIVENESS", "result": "PASS", "reasonCode": null },
    { "type": "FACE_MATCH", "result": "PASS", "reasonCode": null }
  ],
  "nextAction": "CONFIRM_AND_APPLY",
  "completedAt": "2026-08-10T11:31:12+07:00"
}
```

`vhm-dossier-core` lưu reference tới `verificationId/resultVersion` và dữ liệu đã được người dùng xác nhận; không copy toàn bộ raw Canonical Result. Apply phải gửi expected `resultVersion` để tránh dùng kết quả cũ sau retry.

## Thiết kế database

PostgreSQL là system of record cho verification lifecycle. Binary media vẫn nằm trong Private Object Storage do `vhm-dossier-core` sở hữu; database verification chỉ giữ opaque/encrypted reference và metadata tối thiểu.

| **Bảng** | **Mục đích** | **Ràng buộc chính** |
| --- | --- | --- |
| `verification_sessions` | Aggregate root của một attempt OCR/eKYC | Unique idempotency; status/outcome guard; optimistic `row_version` |
| `verification_media_refs` | Media manifest đã authorize/finalize | Unique logical part trong một attempt; không lưu plaintext S3 path/URL |
| `verification_jobs` | Theo dõi orchestration/page job và retry | Unique job unit; lease/retry có giới hạn |
| `provider_attempts` | Một lần gọi operation của provider | Dedupe operation/attempt; provider session ref được mã hóa |
| `verification_checks` | Document/quality/liveness/face/QR/NFC check đã chuẩn hóa | Không lưu raw provider payload |
| `verification_results` | Canonical Result hiện hành | Một result/version hiện hành cho verification; payload PII mã hóa application-layer |
| `verification_history` | Lịch sử state/outcome append-only | Không update/delete trong business flow |
| `outbox_events` | Bảo đảm DB commit và enqueue/event không lệch nhau | Publisher at-least-once; consumer dedupe theo `event_id` |

DDL baseline rút gọn:

```sql
CREATE TABLE verification_sessions (
    verification_id         UUID PRIMARY KEY,
    journey                 VARCHAR(20) NOT NULL CHECK (journey IN ('OCR_ONLY', 'FULL_EKYC')),
    business_type           VARCHAR(30) NOT NULL,
    business_ref            VARCHAR(100) NOT NULL,
    subject_ref_ciphertext  BYTEA,
    consent_ref             VARCHAR(150),
    channel                 VARCHAR(20) NOT NULL,
    document_type           VARCHAR(40),
    status                  VARCHAR(30) NOT NULL CHECK (status IN
                                ('WAITING_MEDIA', 'QUEUED', 'PROCESSING',
                                 'CANCEL_REQUESTED', 'COMPLETED', 'CANCELLED', 'EXPIRED')),
    outcome                 VARCHAR(30),
    attempt_no              INTEGER NOT NULL DEFAULT 1 CHECK (attempt_no > 0),
    retry_of                UUID REFERENCES verification_sessions(verification_id),
    policy_version          VARCHAR(40) NOT NULL,
    result_schema_version   VARCHAR(20) NOT NULL,
    idempotency_key         VARCHAR(100) NOT NULL,
    request_fingerprint     CHAR(64) NOT NULL,
    total_units             INTEGER,
    processed_units         INTEGER NOT NULL DEFAULT 0,
    failed_units            INTEGER NOT NULL DEFAULT 0,
    next_action             VARCHAR(40),
    terminal_reason_code    VARCHAR(80),
    row_version             BIGINT NOT NULL DEFAULT 0,
    expires_at              TIMESTAMPTZ,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at            TIMESTAMPTZ,
    CONSTRAINT uq_verification_idempotency UNIQUE (business_type, idempotency_key),
    CONSTRAINT ck_status_outcome CHECK (
        (status = 'COMPLETED' AND outcome IS NOT NULL) OR
        (status <> 'COMPLETED' AND outcome IS NULL)
    )
);

CREATE INDEX ix_verification_business
    ON verification_sessions (business_type, business_ref, created_at DESC);
CREATE INDEX ix_verification_active_status
    ON verification_sessions (status, updated_at)
    WHERE status IN ('WAITING_MEDIA', 'QUEUED', 'PROCESSING', 'CANCEL_REQUESTED');

CREATE TABLE verification_media_refs (
    media_ref_id             UUID PRIMARY KEY,
    verification_id         UUID NOT NULL REFERENCES verification_sessions(verification_id),
    attempt_no               INTEGER NOT NULL,
    media_id                 VARCHAR(100) NOT NULL,
    media_role               VARCHAR(40) NOT NULL,
    logical_part             VARCHAR(40) NOT NULL DEFAULT 'DEFAULT',
    authorized_ref_ciphertext BYTEA NOT NULL,
    checksum_sha256          CHAR(64) NOT NULL,
    object_version           VARCHAR(200) NOT NULL,
    content_type             VARCHAR(100) NOT NULL,
    size_bytes               BIGINT NOT NULL CHECK (size_bytes > 0),
    finalized_at             TIMESTAMPTZ NOT NULL,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_verification_media UNIQUE (verification_id, media_id),
    CONSTRAINT uq_verification_logical_part
        UNIQUE (verification_id, attempt_no, media_role, logical_part)
);

CREATE TABLE verification_jobs (
    job_id                   UUID PRIMARY KEY,
    verification_id         UUID NOT NULL REFERENCES verification_sessions(verification_id),
    job_kind                 VARCHAR(30) NOT NULL,
    unit_no                  INTEGER NOT NULL DEFAULT 0,
    status                   VARCHAR(20) NOT NULL CHECK (status IN
                                ('PENDING', 'RUNNING', 'RETRY_WAIT', 'SUCCEEDED', 'DEAD')),
    attempt_count            INTEGER NOT NULL DEFAULT 0,
    max_attempts             INTEGER NOT NULL,
    available_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    lease_owner              VARCHAR(100),
    lease_until              TIMESTAMPTZ,
    last_error_code          VARCHAR(80),
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_verification_job_unit UNIQUE (verification_id, job_kind, unit_no)
);

CREATE INDEX ix_job_dispatch ON verification_jobs (status, available_at);

CREATE TABLE provider_attempts (
    provider_attempt_id      UUID PRIMARY KEY,
    job_id                   UUID NOT NULL REFERENCES verification_jobs(job_id),
    provider                 VARCHAR(30) NOT NULL,
    operation                VARCHAR(30) NOT NULL,
    attempt_no               INTEGER NOT NULL,
    provider_session_ciphertext BYTEA,
    provider_request_ref     VARCHAR(150),
    status                   VARCHAR(20) NOT NULL,
    error_class              VARCHAR(30),
    error_code               VARCHAR(80),
    started_at               TIMESTAMPTZ NOT NULL,
    finished_at              TIMESTAMPTZ,
    CONSTRAINT uq_provider_operation_attempt
        UNIQUE (job_id, provider, operation, attempt_no)
);

CREATE TABLE verification_checks (
    check_id                 UUID PRIMARY KEY,
    verification_id         UUID NOT NULL REFERENCES verification_sessions(verification_id),
    check_type               VARCHAR(40) NOT NULL,
    logical_part             VARCHAR(40) NOT NULL DEFAULT 'DEFAULT',
    result                   VARCHAR(20) NOT NULL CHECK (result IN
                                ('PASS', 'FAIL', 'WARN', 'ERROR', 'NOT_APPLICABLE')),
    reason_code              VARCHAR(80),
    restricted_detail_ciphertext BYTEA,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_verification_check
        UNIQUE (verification_id, check_type, logical_part)
);

CREATE TABLE verification_results (
    verification_id         UUID PRIMARY KEY REFERENCES verification_sessions(verification_id),
    result_version           INTEGER NOT NULL,
    schema_version           VARCHAR(20) NOT NULL,
    canonical_payload_ciphertext BYTEA NOT NULL,
    payload_key_version      VARCHAR(40) NOT NULL,
    result_hash              CHAR(64) NOT NULL,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE verification_history (
    history_id               UUID PRIMARY KEY,
    verification_id         UUID NOT NULL REFERENCES verification_sessions(verification_id),
    from_status              VARCHAR(30),
    to_status                VARCHAR(30) NOT NULL,
    outcome                  VARCHAR(30),
    reason_code              VARCHAR(80),
    actor_type               VARCHAR(30) NOT NULL,
    correlation_id           VARCHAR(100) NOT NULL,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_verification_history
    ON verification_history (verification_id, created_at);

CREATE TABLE outbox_events (
    event_id                 UUID PRIMARY KEY,
    aggregate_id             UUID NOT NULL,
    event_type               VARCHAR(80) NOT NULL,
    payload                  JSONB NOT NULL,
    status                   VARCHAR(20) NOT NULL DEFAULT 'NEW',
    available_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at             TIMESTAMPTZ,
    attempt_count            INTEGER NOT NULL DEFAULT 0,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_outbox_publish ON outbox_events (status, available_at);
```

Không dùng `provider_session_id`, `businessRef`, media URL hoặc PII làm metric label. `outbox_events.payload` chỉ chứa ID/reference tối thiểu, không chứa Canonical Result, raw provider payload hoặc object path. Retention/purge chạy theo policy được phê duyệt; `verification_history` giữ tombstone không PII nếu cần chống xử lý lặp sau khi xóa payload.

## Transaction, idempotency và worker

- **Create OCR/eKYC:** trong một transaction ngắn, insert `verification_sessions`, media manifest nếu có, `verification_jobs` và `outbox_events`. Cùng `Idempotency-Key` + fingerprint trả resource cũ; cùng key khác fingerprint trả `409 IDEMPOTENCY_CONFLICT`.
- **Submit media:** HEAD/validate object được thực hiện ngoài transaction. Sau đó lock/CAS `verification_sessions.row_version`, upsert manifest idempotent, kiểm tra đủ logical part, chuyển `WAITING_MEDIA → QUEUED` và ghi outbox trong cùng transaction.
- **Claim job:** queue dùng at-least-once. Worker dedupe theo `job_id`, claim bằng lease/CAS; message trùng không gọi provider lại nếu operation đã có terminal `provider_attempts`.
- **Provider call:** S3/KMS/FPT call luôn nằm ngoài DB transaction. Timeout connect/read, bulkhead, circuit breaker, quota và concurrency được cấu hình riêng theo operation/provider.
- **Persist result:** normalize payload trước; trong transaction ngắn upsert checks/result, cập nhật progress, chuyển `status=COMPLETED`, gán `outcome`, append history và outbox `VerificationCompleted`.
- **Retry:** chỉ retry lỗi network trước khi gửi body hoặc lỗi provider được xác nhận retry-safe. Nếu timeout sau khi body có thể đã tới FPT, reconcile bằng provider session/result API trước; không POST lại mù gây tính phí/kết quả trùng.
- **Tài liệu lớn:** tách `OCR_PAGE` job, giới hạn số trang chạy song song, retry từng trang; aggregator hoàn tất khi mọi unit terminal và cho outcome `PARTIAL` nếu policy cho phép.
- **Dead job:** khi hết recovery budget, worker persist `COMPLETED + PROVIDER_ERROR` và `nextAction=RETRY`; DLQ chỉ phục vụ vận hành, không để client polling vô hạn.
- **Cancel:** `WAITING_MEDIA/QUEUED` chuyển `CANCELLED` ngay. Khi `PROCESSING`, chuyển `CANCEL_REQUESTED`; worker không phát sinh call mới, kết thúc call đang chạy rồi bỏ result và chuyển `CANCELLED`.

## Cấu trúc implementation

```text
vhm-verification-service
├── api/internal            # REST contract, authn/z, validation, error mapping
├── application             # create/submit/status/result/cancel/retry use cases
├── domain                  # journey, lifecycle, policy, outcome, invariants
├── worker                  # OCR/eKYC/page worker, aggregator, reconciliation
├── provider/spi            # provider-neutral ports
├── provider/fpt            # FPT session/OCR/liveness/QR/NFC adapters
├── media                   # validate/read opaque authorized media reference
├── persistence             # PostgreSQL repositories, locking, migrations
├── messaging               # outbox publisher, queue consumer, DLQ
└── security-observability  # KMS/secrets, masking, audit, metrics, tracing
```

API và Worker là hai deployment độc lập nhưng dùng chung codebase/domain contract. API scale theo request rate; Worker scale theo queue lag nhưng luôn bị chặn bởi provider quota. Outbox Publisher và Reconciliation có thể chạy trong Worker deployment với leader election hoặc consumer partitioning.

**Baseline bảo mật và vận hành:**

| **Nhóm** | **Implementation bắt buộc** |
| --- | --- |
| Network/IAM | Service private; mTLS + service token; S3 read exact object; egress allowlist chỉ tới Object Storage, KMS/Secret Manager và endpoint FPT |
| Secret/PII | API key/session reference trong Secret Manager/KMS; TLS in transit; DB/S3 encryption at rest; application-layer encryption cho Canonical Result; masking theo role |
| Input | Giới hạn type/size/page/duration; kiểm tra checksum, MIME + magic bytes, object version; malware scan nếu policy tài liệu yêu cầu |
| Logging/Audit | Không log body/media/PII/credential; audit create, submit, query result, retry, cancel, state/outcome transition và apply result |
| Resilience | Timeout, exponential backoff + jitter, circuit breaker, bulkhead, rate limit, queue/DLQ, bounded reconciliation; không polling provider vô hạn |
| Metrics | API latency/error, queue depth/oldest age, processing duration, retry/DLQ, provider latency/error/quota, outcome count và stuck session; không dùng high-cardinality/PII label |
| Availability | API tối thiểu 2 replicas; Worker graceful shutdown/lease recovery; PostgreSQL Multi-AZ/PITR; queue durable; runbook provider outage/backlog recovery |

**Thứ tự triển khai đề xuất:**

1. Xây API/status model, PostgreSQL migration, idempotency, outbox, queue, worker skeleton, security/audit và mock Provider Adapter.
2. Triển khai `OCR_ONLY` cho CCCD/GPLX/Hộ chiếu qua FPT `/session/init` → `/ocr`, Canonical Result và polling end-to-end.
3. Bổ sung Document OCR cho PDF/tài liệu nhiều trang sau khi chốt provider, input limit, SLA và sync/async contract.
4. Triển khai `FULL_EKYC` `/session/init` → `/ocr` → `/face/liveness`; chỉ bật QR/NFC theo channel/policy đã kiểm thử. FPT public API hiện mô tả NFC `check_chip` cho Android/iOS, không mặc định bật cho Web.
5. Load/chaos/security test; chốt retry budget, provider quota, retention, masking và alert threshold trước production.

Các điểm còn phải xác nhận với FPT trước khi code production: SDK có expose raw artifact trước khi gửi provider hay không; liveness mode/media requirement theo Mobile/Web; QR/NFC capability theo channel; giới hạn và SLA của AI Read/Document OCR; provider idempotency, timeout-unknown handling, result retention và quota. Callback chỉ bổ sung khi có contract xác thực/dedupe rõ ràng và đi qua Callback Ingress, không mở public service private.

`vhm-verification-service` chịu trách nhiệm capability OCR/eKYC và dữ liệu verification. Quyết định sử dụng kết quả, cập nhật khách hàng và phê duyệt hồ sơ NOXH vẫn thuộc `vhm-dossier-core`.

