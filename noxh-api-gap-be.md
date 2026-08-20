# NOXH — Danh sách API/field FE còn thiếu, gửi BE

> Cập nhật 2026-08-19 · Đối chiếu `vhm-agent-api-openapi.yaml` + `openapi-noxh.yaml` với code `apps/agent-v1/src/modules/SocialHousing/`.
> Phạm vi: UC-01…UC-05, UC-07, UC-08. Chỗ nào FE đã tự đặt tạm đều ghi rõ.

---

## Câu hỏi chặn — v1 hay v2?

FE gọi `/api/v1/social-housing`, nhưng khi bật cờ `AUTH-02` gateway đổi sang `/api/v2/social-housing`. Bản v2 (`openapi-noxh.yaml`) **thiếu** `PUT /registrations/{id}/owner`, `source`, `owner`, `documents[].fileName` — những thứ bản v1 đã có. Nhờ BE xác nhận v2 có được đồng bộ không, vì thiếu `/owner` là UC-03 hỏng ngay khi bật cờ.

---

## UC-01 — Tạo KH + phân phối Contact

1. Đã có API lấy hồ sơ NOXH liên kết khách hàng để hiển thị ở màn Chi tiết khách hàng (`GET /registrations/by-contact`) nhưng **thiếu field `owner`** để hiển thị "Sale hỗ trợ" ở khối hồ sơ NOXH. Hiện chỉ trả `agentId`/`agentName` nên mọi dòng đều ra badge "Chờ giao việc".

2. **Bổ sung API lấy hồ sơ NOXH liên kết nhu cầu** để hiển thị ở màn Chi tiết nhu cầu — tương tự màn khách hàng đã có. Đề xuất `GET /registrations/by-inquiry?inquiryId={id}&page&pageSize`, dùng lại y nguyên response + phân quyền + phân trang của `by-contact`. *(Hiện FE tạm gọi `GET /registrations?linkedInquiryId=` — BE không nhận param này nên card đang liệt kê nhầm hồ sơ của nhu cầu khác.)*

3. **Bổ sung 2 field lỗi** trên hồ sơ để hiện banner: `formSubmissionError` (AF-01 — tạo Yêu cầu tư vấn thất bại) và `contactCreationError` (AF-02 — tạo Contact/Inquiry thất bại). Kèm 1 câu hỏi: khi tạo Yêu cầu tư vấn lỗi thì hồ sơ NOXH **có tồn tại** bên Agent không? Nếu không thì bỏ luôn `formSubmissionError`.

## UC-02 — Đồng bộ LDP Market

1. **Bổ sung `linkedInquiryId`** trên hồ sơ — để link sang màn Nhu cầu.

2. **Bổ sung `linkedInquirySummary`** `{ name, code, serviceTypeLabel, createdAt }` nhúng thẳng vào `GET /registrations/{id}`. Lý do phải nhúng: role NOXH (900/903/906/909/910) không có quyền xem Inquiry, FE gọi API Inquiry riêng sẽ bị 403 đúng với người cần thấy nhất.

3. **Bổ sung `linkedContactId`** — để link sang màn Khách hàng Vinhomes.

4. **Bổ sung `ocrStatus`** cấp hồ sơ (`not_ocr | processing | done | error`) — để hiện cột + badge Trạng thái OCR. Đây là phần *hiển thị* trạng thái đã đồng bộ sẵn từ Market, khác với phần *chạy* OCR ở UC-05.

## UC-03 — Sale phụ trách

1. **Bổ sung API lấy danh sách Sale được phép gán** cho 1 hồ sơ. Hiện FE tự lọc theo team của người thao tác, không phải rule của BE, nên user chọn xong mới ăn `403 11018 DOSSIER_OWNER_TEAM_MISMATCH`. *(Nếu không làm API này thì tối thiểu bổ sung `owner.teamId` + `owner.teamName` để FE lọc đúng.)*

2. **`PUT /registrations/{id}/owner` có nhận `comment` không?** Modal có ô Ghi chú 1000 ký tự nhưng contract chỉ có `owner` nên FE đang không gửi lên.

3. *(nice-to-have)* `owner.avatar` / `owner.title` / `owner.isActive` — chỉ để hiển thị đẹp.

## UC-04 — Tiến độ hoàn thành

1. Không cần API mới, FE tự tính từ `documents[]`. Chỉ 1 câu hỏi: **BE trả gì khi hồ sơ không match được checklist nào?** *(Nếu BE muốn là nguồn chính thì trả `progressPercent` / `completedDocCount` / `totalRequiredDocCount`, FE sẽ đổi sang dùng số của BE.)*

## UC-05 — OCR bộ hồ sơ (mốc 15/9)

1. **Chưa có API nào.** FE đang tự đặt tạm `POST /registrations/{id}/ocr` (chạy OCR) + `GET /registrations/{id}/ocr-status` (poll tới khi xong). Nhờ BE chốt endpoint + body + response thật.

## UC-07 — Cấu hình quản lý hồ sơ

1. CRUD biểu mẫu / bộ biểu mẫu / phân quyền dự án **đã đủ**. Chỉ thiếu `typeCode` — xem UC-08 mục 2.

## UC-08 — Enhancement

1. **Bổ sung `registrationTypeCode`** (Mã đối tượng đăng ký, BR-02) — hỏi BE để ở cấp hồ sơ hay trả kèm trong property option `subject_group`.

2. **Bổ sung `typeCode`** (Mã giấy tờ, BR-04) ở **cả hai** nơi: `documents[]` của hồ sơ *và* `documentTemplates[]` của bộ biểu mẫu. Thiếu vế bộ biểu mẫu thì màn Tạo mới không hiện được mã. Kèm theo: màn cấu hình checklist cần cho **nhập** field này.

3. **Bổ sung `validationGuide`** (hướng dẫn **thẩm định**, cho người duyệt, BR-05). Khác `description` hiện có (hướng dẫn **chuẩn bị**, cho đại lý) — hỏi BE thêm field mới hay tách 2 nghĩa.

4. **Xác nhận dùng `GET /registrations/statistics` thay cho việc đếm hồ sơ theo tab (BR-07)** — nó đã trả `byQueue[].count` với đúng bộ lọc FE gửi. Chỉ cần biết param `queue` có bắt buộc ≥1 giá trị không. *(FE đang gọi tạm `GET /registrations/queue-count` — BE không có endpoint này.)*

5. **Phiếu tiếp nhận tự sinh checklist (BR-08, mốc 15/9)** — `GET /registrations/{id}/download` hiện có `type=CONTRACTS | ATTACHMENTS`; cần thêm type mới hay endpoint riêng?

6. **Thêm giá trị "Đơn thân" vào property option `marital_status`** (BR-10, mốc 15/9) — chỉ là data config.

7. **BE mask `applicant.phone` ở `GET /registrations/{id}`** + trả cờ cho biết field nào được phép mở (BR-00). Hiện FE tự che khi hiển thị nhưng **số đầy đủ vẫn nằm nguyên trong payload**, mở DevTools là đọc được — chỉ BE mask mới chặn được.

---

## 3 endpoint FE đang gọi mà không thấy trong contract

Ba cái này FE gọi thật hằng ngày trên production nên nhiều khả năng bản contract BE gửi bị thiếu, nhưng cần xác nhận — nếu BE thật sự không có thì nút "Rút hồ sơ" và "Yêu cầu bản cứng" đang lỗi trên production.

1. `POST /registrations/{id}/withdraw` — nút "Rút hồ sơ", và fallback khi xoá hồ sơ nháp trả `11010`.
2. `POST /registrations/{id}/request-hardcopy` — nút "Yêu cầu bản cứng". Contract chỉ có `submit-hardcopy` + `confirm-hardcopy`.
3. `POST /registrations/{id}/documents/{documentId}/reviews` — contract chỉ khai `GET`. **FE hiện không còn dùng** (luồng thẩm định đã chuyển sang `POST /{id}/documents/decision`) ⇒ BE xác nhận không làm thì FE xoá hẳn.
