# Nhật ký cập nhật tài liệu OCR/eKYC

**Ngày cập nhật:** 10/08/2026  
**Tài liệu chính:** `noxh-ocr-ekyc.md`  
**Phạm vi:** `Vấn đề 6: Lựa chọn nền tảng tích hợp OCR tập trung`

## 1. Tài liệu FPT đã đối chiếu

- [Hướng dẫn cấu hình eKYC Portal](https://docs-vision.fpt.ai/category/detailed-instructions-in-ekyc-portal-configuration/)
- [System Integration](https://docs-vision.fpt.ai/category/system-integration/)
- [So sánh các phương thức tích hợp](https://docs-vision.fpt.ai/ekyc/III-integration/III-0-so-sanh/)
- [Kiến trúc SDK qua Proxy Server](https://docs-vision.fpt.ai/ekyc/III-integration/III-1-SDKs/kien-truc-tich-hop/)
- [FPT Web SDK](https://docs-vision.fpt.ai/en/ekyc/III-integration/III-1-SDKs/web-sdk/)
- [FPT eKYC backend API](https://docs-vision.fpt.ai/ekyc/III-integration/III-2-APIs/a-APIs%20of%20eKYC%20Flows/APIs-in-update-information-flow/)

Các điểm đã xác nhận:

- FPT eKYC SDK hỗ trợ capture/quality guidance cho giấy tờ định danh và liveness trên Mobile/Web.
- SDK cho phép cấu hình endpoint proxy cho init session, OCR và liveness; request/response đi qua Proxy Server của khách hàng.
- FPT eKYC `/ocr` công bố các loại `idr`, `passport`, `dlr`; chưa có căn cứ dùng eKYC SDK như kênh OCR tài liệu hồ sơ tổng quát.
- Provider/model/API contract cho giấy đăng ký kết hôn, giấy chứng nhận hộ nghèo/cận nghèo và các tài liệu nghiệp vụ vẫn cần xác nhận với FPT.

## 2. Quyết định kiến trúc đã chốt

Áp dụng kiến trúc hybrid trong cùng `vhm-verification-service`:

```text
OCR tài liệu hồ sơ
→ Presigned PUT qua vhm-media-service
→ OCR queue/worker
→ Document OCR API/model
→ polling Canonical Result

EKYC
→ FPT SDK trên Mobile/Web
→ Streaming Proxy Ingress
→ eKYC SDK Proxy trong vhm-verification-service
→ FPT eKYC Backend
→ lưu Canonical Result khi response đi qua Proxy
```

### Lý do chọn

- eKYC là luồng tương tác thời gian thực; SDK cung cấp capture guidance, quality check, liveness, face matching và trải nghiệm tốt hơn API-only.
- Proxy giữ FPT credential ở server, cho phép VHM kiểm soát data path và lưu kết quả mà SDK vẫn nhận response đúng contract.
- PDF/tài liệu hồ sơ lớn không thuộc phạm vi eKYC SDK đã công bố; Presigned PUT tránh truyền binary lớn qua BFF và phù hợp xử lý OCR bất đồng bộ.
- OCR và eKYC vẫn dùng chung `verificationId`, status/result API, Result Normalizer, Canonical Result và Verification Database; không cần tách thành hai service.

## 3. Luồng OCR tài liệu hồ sơ

```text
Mobile/Web
→ vhm-agent-api lấy upload slot
→ vhm-media-service trả mediaId + Presigned PUT URL
→ Mobile/Web PUT trực tiếp Private Object Storage
→ finalize mediaId
→ vhm-dossier-core authorize tạo OCR
→ vhm-verification-service persist QUEUED + outbox
← 202 + verificationId + statusUrl

OCR Worker
→ lấy read grant cho exact media version
→ đọc document
→ Provider Adapter gọi Document OCR API/model
→ normalize và lưu COMPLETED + outcome + Canonical Result
```

- Một file PDF/tài liệu là một logical document và một `mediaId`.
- Trường hợp contract cần nhiều artifact, ví dụ hai mặt CCCD, OCR request dùng manifest nhiều `mediaId` nhưng vẫn chỉ có một job và một Canonical Result.
- Không tách page/batch job, không có `OCR_PAGE`, progress theo trang hoặc outcome `PARTIAL`.
- Chỉ enable `documentType` đã chốt provider/model, input contract, SLA và error contract.

## 4. Luồng eKYC qua SDK Proxy

```text
Mobile/Web
→ vhm-agent-api
→ vhm-dossier-core authorize
→ Verification API tạo verificationId + WAITING_CAPTURE
← SDK bootstrap: proxy endpoints + short-lived token

FPT SDK
→ Streaming Proxy Ingress
→ eKYC SDK Proxy
→ FPT /init_session
← session-id
→ FPT /ocr
← OCR result
→ FPT /face/liveness
← liveness + face-match result

eKYC SDK Proxy
→ normalize checks
→ lưu COMPLETED + VERIFIED/REJECTED/NEED_RETRY/PROVIDER_ERROR
```

- eKYC không dùng `vhm-media-service`, Object Storage, queue hoặc worker.
- SDK cần response đồng bộ của FPT để tiếp tục UI flow; Proxy không thay response bằng `202`.
- Streaming Proxy Ingress là public boundary; `vhm-verification-service` vẫn private.
- Proxy token bind `verificationId/attempt/subjectRef/allowedOperations/expiresAt`; không chứa FPT API key hoặc provider session.
- Proxy stream multipart end-to-end, không buffer toàn bộ ảnh/video, không log body và không retry mù sau khi bắt đầu gửi body.
- Retry eKYC tạo attempt, token và FPT session mới.

## 5. Phân định trách nhiệm

- **Mobile/Web:** uploader của VHM cho OCR; FPT SDK cho eKYC; query/confirm kết quả qua Application API.
- **`vhm-agent-api`:** xác thực/routing Application API và authorize OCR upload/finalize; không giữ FPT credential.
- **Streaming Proxy Ingress:** authenticate proxy token, áp dụng body-size/timeout/concurrency và stream request tới private service.
- **`vhm-dossier-core`:** authorize create/query/apply journey và cập nhật hồ sơ từ kết quả đã xác nhận.
- **`vhm-media-service`:** chỉ phục vụ OCR upload/finalize, immutable metadata và read grant.
- **`vhm-verification-service`:** Verification API, OCR Worker, eKYC SDK Proxy, Provider Adapter, Result Normalizer, Decision Mapper và Canonical Result.
- **Verification Database:** session; OCR media snapshot/job; eKYC proxy token hash/provider session; provider attempts, checks, result và history.

## 6. Lifecycle và API đã đổi

- OCR: `QUEUED → PROCESSING → COMPLETED`.
- eKYC: `WAITING_CAPTURE → PROCESSING → COMPLETED`.
- `CANCELLED/EXPIRED` kết thúc không có outcome; `outcome` chỉ có khi `COMPLETED`.
- Bỏ `WAITING_MEDIA`, eKYC media submission API và eKYC worker job.
- Bổ sung SDK bootstrap cùng các proxy route `/init-session`, `/ocr`, `/face/liveness`.
- `verification_jobs` và `verification_media_refs` chỉ dùng cho OCR.
- Bổ sung `ekyc_proxy_sessions`; `provider_attempts` bind trực tiếp `verificationId`, còn `job_id` chỉ có với OCR.

## 7. Các nội dung khác đã hoàn thành

- Tách toàn bộ Vấn đề 6 sang `noxh-ocr-ekyc.md`; `noxh.md` chỉ giữ quyết định và link tài liệu chi tiết.
- Chuẩn hóa tên service: `vhm-agent-api`, `vhm-dossier-core`, `vhm-media-service`, `vhm-verification-service`.
- Giữ `vhm-verification-service` làm một service; các box API/worker/proxy/adapter trong Mermaid chỉ là module/workload nội bộ.
- Bỏ state diagram, phần công nghệ/cấu trúc source code và mục bảo mật/vận hành riêng theo review.
- Bổ sung API contract, Canonical Result, PostgreSQL DDL baseline, idempotency, outbox, retry/cancel và kiểm thử.
- Không trả raw provider payload qua Application API; SDK Proxy chỉ giữ response shape cần thiết cho SDK.

## 8. Việc chưa thực hiện/cần xác nhận

- Chưa render/tạo lại ảnh từ Mermaid; chỉ thực hiện một lần sau khi nội dung được chốt.
- Cần chốt provider/model và API contract cho các tài liệu nghiệp vụ ngoài `idr/passport/dlr`.
- Cần xác nhận SDK version, proxy endpoint/header contract, ảnh/video limit, timeout và liveness mode với FPT.
- Cần chốt Streaming Proxy Ingress body-size, idle/request timeout, backpressure, connection reuse và autoscaling.
