# BDSKD-9533 — Mapping các phần PRD với bảng DB

Ngày kiểm tra: **05/10/2026**. Tài liệu đầu vào: [Phân tích PRD](BDSKD-9533-phan-tich-PRD.md). Đối chiếu code commit `31444ce6` và **kết nối trực tiếp, chỉ đọc** DB staging `vhmmarket_db`, schema `cobroker_db`, bằng cấu hình handoff. Không ghi dữ liệu vào DB.

Đã kiểm tra danh sách bảng, metadata cột, constraint/index và thống kê mapping. Phiên kết nối dùng transaction read-only và rollback. Chưa kiểm tra schema/service CMS, profile-mw, Housing hoặc SAP. Các số lượng dưới đây chỉ là snapshot staging tại thời điểm đọc.

**Kết luận:** có nguồn hồ sơ sale đại lý, xác thực, đăng ký dự án, bán căn theo đại lý, room và outbox. Chưa có bảng chu kỳ/chính sách chu kỳ; chưa xác nhận được ngày bắt đầu bán; nguồn GD cá nhân đã được người dùng xác định là vhm-sale-pipeline qua Kafka, payload cụ thể cần thống nhất. Toàn bộ tên `sale_cycle_*` trong tài liệu này là **đề xuất trong thiết kế kỹ thuật hiện có**, chưa tồn tại trong schema đã đọc.

## 1. Mapping theo từng phần của tài liệu PRD

| Phần trong phân tích PRD | Bảng hiện có liên quan | Dữ liệu dùng được | Dữ liệu còn thiếu / bảng đề xuất |
| --- | --- | --- | --- |
| **1. Mục tiêu theo dõi từng sale** | `agency_cobroker`, `cobroker_profiles`, `agency_profiles` | Tài khoản Agent, hồ sơ người, đại lý và vai trò | `sale_cycle_profiles` để lưu đối tượng theo dõi; kết quả cần policy + fact + period |
| **2. Chính thức → thử thách → đề xuất chấm dứt** | Không có bảng chu kỳ chuyên biệt | Profile/link chỉ phản ánh hoạt động/xác thực/việc làm | `sale_cycle_period` lưu kỳ; `sale_cycle_profiles` lưu phân loại và con trỏ kỳ. Không dùng `status` profile/link thay giai đoạn |
| **3. Ví dụ và chuyển kỳ theo ngày/GD** | `identity_verification_history`, `cobroker_applicant_agencies_submission`, `sale_batch_units` chỉ hỗ trợ đối chiếu đầu vào | Lịch sử xác thực và tình trạng bán căn | Ngày bán phải được xác nhận; GD cá nhân cần `sale_cycle_transactions`; policy/rule/period/task phục vụ tính và chuyển kỳ |
| **4. Chính sách/Kinh doanh xem báo cáo, thiết lập cấu hình** | Profile/link/đại lý cung cấp danh tính và tổ chức | Họ tên, Agent ID, mã hồ sơ, đại lý/vùng | `sale_cycle_profiles`, `sale_cycle_period` cho báo cáo; `sale_cycle_policy`, `sale_cycle_policy_rule` cho cấu hình/history |
| **4. Admin đại lý/QLKD lọc và xuất Excel** | `agency_profiles`, `agency_cobroker`, `cobroker_profiles`; tổ chức O2O/Tự doanh từ nguồn ngoài | Scope đại lý và dữ liệu hồ sơ; cây tổ chức không nằm trong các bảng này | Query báo cáo dùng cùng scope/filter với export; `audit_logs` là đề xuất lưu lần xuất |
| **4. Sale nhận nhắc và kết quả** | `notification_outbox` | Loại thông báo, đối tượng, giờ gửi, payload | `sale_cycle_notification_delivery` giữ lịch sử/dedupe; thêm handler và receiver theo Agent ID |
| **5. Hồ sơ/quan hệ sale đại lý** | `cobroker_profiles`, `agency_cobroker`, `agency_profiles` | Mapping chi tiết ở mục 2 | O2O/Tự doanh và nối lịch sử khi chuyển đại lý chưa được xác minh |
| **5. Xác thực/ngày bắt đầu bán** | Submission + `identity_verification_history`; `import_job`, `import_job_item` tham khảo hạ tầng import | Submission, sự kiện xác thực, dữ liệu import/lỗi từng dòng | Chưa có cột ngày bắt đầu bán chuyên biệt; đề xuất lưu trên hồ sơ chu kỳ, audit sửa ngày |
| **5. GD bán và hạn mức căn** | `sale_batch_units`, `agency_tax_codes`, `sale_batches`, `sale_batch_agencies`, `user_registered_scope`, `sale_batch_agency_rooms`, `room_ledger` | Căn bán theo đại lý; sale hợp lệ theo dự án; room/ledger | Chưa có attribution GD cá nhân; thêm kết quả chu kỳ vào điều kiện đếm sale cấp room |
| **6. SRS bổ sung — ngày, báo cáo, Excel, cấu hình, noti** | Dùng lại các nguồn hồ sơ/room/outbox bên trên | Nền dữ liệu của sáu US | Mapping theo US ở mục 6; yêu cầu SRS bổ sung không có nghĩa DB đã hỗ trợ |
| **7. Điểm cần chốt** | Tax code/căn/xác thực/history giúp đối soát, không tự quyết định luật | Truy nguyên thông tin đang có | GD hợp lệ, ngày nghiệp vụ, reset policy, hồi tố, room đã cấp và chuyển đại lý vẫn cần xác nhận |
| **8. Phạm vi bàn giao** | Không có bảng tương ứng riêng | Các bảng nền có thể tái sử dụng | Thứ tự triển khai xem [DB và lộ trình](BDSKD-9533-db-va-lo-trinh.md); scope tháng 10 cần chính sách ban đầu và nguồn GD đã chốt |

## 2. Hồ sơ, tài khoản và tổ chức — báo cáo/US-01, US-02

### 2.1. Cột dữ liệu thực tế

| Thông tin nghiệp vụ | Bảng.cột hiện có | Cách hiểu / giới hạn |
| --- | --- | --- |
| Tài khoản Agent / ID hệ thống trên báo cáo | `agency_cobroker.agent_profile_id` (`varchar`, nullable) | Khóa tài khoản theo quan hệ đại lý; không thay bằng ID UUID hồ sơ |
| ID hồ sơ người | `cobroker_profiles.id` (`uuid`) | Join từ `agency_cobroker.cobroker_profile_id` |
| IAM Account | `cobroker_profiles.account_id` (`uuid`, nullable, unique) | Định danh IAM; khác với Agent ID |
| Mã hồ sơ co-broker | `cobroker_profiles.cobroker_id` (`varchar`, unique) | Không mặc định là mã nhân viên CMS; cột DB cho phép NULL dù model khai báo bắt buộc |
| Họ tên / mã định danh | `cobroker_profiles.full_name`, `identity_no` | Phục vụ hiển thị/tìm kiếm; không dùng tên/CCCD làm khóa nối GD |
| Vai trò tại đại lý | `agency_cobroker.role_type` (`varchar[]`) | Có thể chứa đồng thời `SALE_ADMIN`, `SALE_MEMBER`; dùng `ANY`, không so sánh scalar |
| Trạng thái hồ sơ | `cobroker_profiles.status` | Code có DRAFT/ACTIVE/INACTIVE/FROZEN/REQUEST_UPDATE; không mapping trực tiếp toàn bộ nhãn SRS bằng một cột này |
| Trạng thái việc làm | `agency_cobroker.status`, `lock_reason`, `locked_at` | ACTIVE/INACTIVE theo đại lý; khác với trạng thái chu kỳ |
| Đại lý | `agency_cobroker.agency_profile_id` → `agency_profiles.id` | `agency_profiles.brand_name`, `agency_code`, `legal_name` cho hiển thị |
| Tổ chức tenant | `agency_cobroker.organization_id`, `agency_profiles.organization_id` (`int4`) | Phải kiểm tra scope người xem; có ID không chứng minh user có quyền |
| Vùng đại lý | `agency_profiles.region_team_id` (`int4`), `region_name` | Snapshot vùng phẳng; không phải cây tổ chức đầy đủ |
| Team đại lý bên profile-mw | `agency_profiles.team_id` (`int4`, nullable) | Dấu vết liên kết; không thay khóa domain `agency_profiles.id` |
| Region/department của hồ sơ | `cobroker_profiles.region_id`, `department_id` (`varchar`) | Code resolve tên ở profile-mw; khác kiểu/key với `region_team_id`, không join trực tiếp bằng cast |
| Chức vụ | `agency_cobroker.employment_metadata ->> 'position'` | JSONB, free text theo model; chưa chứng minh là cấp bậc/quản lý trong org chart SRS |
| Mã nhân viên, Ngày BC, Miền/Phòng/Đội O2O/Tự doanh | Chưa xác nhận nguồn/cột | Cần hợp đồng CMS/profile; bảng `account` trong core không đủ chứng minh đã có danh sách này |

### 2.2. Các khóa join đã đối chiếu

```text
agency_cobroker.cobroker_profile_id        → cobroker_profiles.id
agency_cobroker.agency_profile_id          → agency_profiles.id
user_registered_scope.username            → agency_cobroker.agent_profile_id
cobroker_profiles.cobroker_applicant_agencies_submission_id
                                          → cobroker_applicant_agencies_submission.id
identity_verification_history.submission_id
                                          → cobroker_applicant_agencies_submission.id
```

Đây là **quan hệ logic được code sử dụng**. Metadata constraint của các bảng hồ sơ/link/xác thực đã kiểm tra không có FK cưỡng chế các liên kết trên. Không gọi các mũi tên này là FK vật lý.

Submission còn có `cobroker_profile_id` và `agent_profile_id`: dùng để đối soát các submission theo người/tài khoản, không lấy mọi row submission làm một sale mới. Con trỏ submission trên profile không tự chứng minh là lần xác thực đầu tiên hoặc submission thuộc đúng lần làm việc tại đại lý đang xét.

Ràng buộc hiện có đáng chú ý:

- `uq_agency_cobroker_agent_profile_id`: UNIQUE `agent_profile_id` với điều kiện IS NOT NULL; nhiều NULL vẫn được phép.
- `uq_agency_cobroker_active_profile`: mỗi `cobroker_profile_id` tối đa một link ACTIVE.
- `uq_agency_cobroker_agency_profile`: UNIQUE `(agency_profile_id, cobroker_profile_id)`.
- `user_registered_scope` có PK `(username, registration_type, scope_type, scope_id)`: một sale có thể có nhiều scope; join thẳng vào báo cáo dễ nhân dòng.

**Dữ liệu thực tế:** 1.006 link có `SALE_MEMBER`, 92 link có `SALE_ADMIN`; các tập có thể giao nhau. Không thấy Agent ID trùng hoặc SALE_MEMBER thiếu/trống Agent ID. Có **2 link trỏ tới profile không tồn tại**, 0 link trỏ tới đại lý không tồn tại. Cần đối soát 2 link này trước backfill; `INNER JOIN` sẽ làm mất chúng khỏi danh sách mà không báo lý do.

Báo cáo chu kỳ vẫn theo dõi sale ngưng hoạt động theo SRS. Không tái sử dụng nguyên predicate room chỉ lấy link ACTIVE cho tập đối tượng báo cáo.

## 3. Ngày bắt đầu bán và xác thực — US-06

| Bảng/cột thực tế | Có thể dùng cho | Không được tự suy ra |
| --- | --- | --- |
| `cobroker_applicant_agencies_submission.identity_verification_progress` (`jsonb`) | Tiến trình các leg OCR/eKYC | Một leg DONE chưa tự chứng minh toàn bộ điều kiện Đã xác thực đã hoàn tất |
| Submission `status`, `ekyc_verified`, `manual_decisions`, `approval_metadata`, `approve_at` | Đối soát trạng thái và lần duyệt | Không lấy mọi `approve_at` làm ngày bắt đầu mới |
| `identity_verification_history.submission_id`, `stage`, `leg`, `event`, `event_at`, `value_before`, `value_after` | Truy nguyên sự kiện hoàn thành/duyệt/thay đổi theo submission | Không lấy `MIN(event_at)` của mọi event hoặc mọi APPROVED làm ngày bắt đầu bán |
| `cobroker_profile_history.cobroker_profile_id`, `event`, `field`, `value_before`, `value_after`, `event_at` | Đối soát thay đổi hồ sơ/hoạt động | Lịch sử hồ sơ không thay bảng kỳ hoặc lịch sử ngày bán |
| `agency_cobroker.employment_metadata ->> 'laborContractStartDate'` | Ngày bắt đầu HĐLĐ theo model | **Không phải ngày bắt đầu bán đã được xác nhận** |
| `import_job`, `import_job_item` | Tham khảo hạ tầng import, status/lỗi từng dòng | Chưa có bằng chứng type/handler import ngày bắt đầu bán cho 9533 đã được triển khai |

Không thấy cột ngày bắt đầu bán chuyên biệt trên `cobroker_profiles` và `agency_cobroker`. JSONB employment trong dữ liệu đang có các key `avatar`, `contactAddress`, `dateOfBirth`, `laborContractStartDate`, `position`; chưa thấy key ngày bắt đầu bán. Kết luận này giới hạn ở cột/key đã kiểm tra và model hiện hành.

History có cả APPROVED của leg OCR và EKYC, EKYC_COMPLETED, OCR_RECONCILED và các sự kiện FORCE_REJECT. Vì vậy chỉ đếm/lấy MIN(APPROVED) chưa phân biệt được xác thực lần đầu, duyệt một leg và xác thực lại. Cần đối chiếu luồng conclude/activation trước khi chọn mốc; tài khoản cũ vẫn dùng file Chính sách cung cấp theo SRS.

**Lưu ý kiểu ngày:** `submission.approve_at`, `submission.created_at` và `cobroker_profiles.created_at` là `timestamp without time zone` trong DB; `identity_verification_history.event_at` là `timestamptz`. Phải xác nhận timezone của timestamp cũ trước khi đổi thành ngày nghiệp vụ. Ngày bắt đầu bán đề xuất lưu kiểu `date`; không lấy ngày tạo hồ sơ để lấp thiếu dữ liệu.

Đích đề xuất: `sale_cycle_profiles.start_date`, `start_date_source`, `start_date_override`; ghi thay đổi vào `audit_logs`, enqueue `async_job` để tính lại. Các cột ngày mới nằm trên sale_cycle_profiles; audit_logs và async_job đã có, cần thêm action/handler.

## 4. Giao dịch được tính chỉ tiêu — US-01 và mục 7 PRD

| Bảng/cột thực tế | Ý nghĩa và mapping |
| --- | --- |
| `sale_batch_units.id`, `batch_id`, `unit_id`, `sap_id` | Row căn trong đợt bán; `sap_id` đối chiếu house unit code từ Housing; không phải khóa GD cá nhân |
| `sale_batch_units.sale_status` | Tình trạng căn FOR_SALE/BOOKING/SOLD do processor hiện hành cập nhật |
| `sold_by_agency_profile_id` → `agency_profiles.id` | Đại lý được ghi nhận bán |
| `sold_by_agency_tax_code` | MST đại lý từ nguồn GD, có thể đối chiếu `agency_tax_codes.tax_code` |
| `agency_tax_codes.agency_profile_id`, `tax_code`, `status`, `effective_from`, `inactivated_at` | Mapping MST về đại lý và dấu vết hiệu lực; không mapping từ MST ra đúng một sale |
| `sale_batch_units.signed_at`, `sold_at` | Mốc ký/ghi nhận bán theo luồng hiện hành; chưa xác nhận mốc tính chỉ tiêu TTĐC/TTKQ |
| `allocated_to_agency_profile_id`, `distributed_agency_profile_id`, `allocated_by`, `allocated_by_item_id` | Phân bổ/phân phối/thao tác căn; không chứng minh ai là sale hưởng GD |
| `sale_batch_unit_history` | Lịch sử thay đổi căn; không mặc định là ledger GD cá nhân |

DTO `SaleOrderChangeEvent.Data` hiện có `transCode`, `transStatus`, `status`, `agencyTaxCode`, `actualReservationSignTime`, `actualContractSignTime`; **chưa map sale/Agent ID**. Envelope `objectId` là ID `sale_orders` bên Housing, không được mặc định là một khóa GD chống trùng xuyên TTĐC/TTKQ cho 9533.

Processor hiện tại ánh xạ ZS03/ZS04/ZS07 và nhiều trạng thái SAP thành SOLD để xử lý bán căn/room. SRS 9533 yêu cầu GD xác nhận TTĐC/TTKQ: cần xác nhận mapping riêng, không dùng toàn bộ STATUS_MAP hiện tại làm điều kiện đếm chỉ tiêu. `signedAt()` của DTO ưu tiên mốc nhỏ hơn giữa ký cọc và HĐMB; đây cũng chưa phải luật ngày tính chỉ tiêu 9533 đã được duyệt.

Snapshot có 243 row căn SOLD, 33 BOOKING, 12.285 FOR_SALE. **243 row SOLD không suy ra 243 GD đủ điều kiện hoặc số GD theo từng sale**; một căn có thể xuất hiện ở các đợt khác nhau và bảng không có sale attribution.

**Nguồn triển khai đã xác nhận từ người dùng:** vhm-sale-pipeline xác định GD của sale và phát Kafka; core nhận/lưu fact. Luồng Housing và các cột căn bên trên là hiện trạng module căn, không phải nguồn tích hợp GD cá nhân cho 9533.

Đích đề xuất: `sale_cycle_transactions`, cần pipeline cung cấp tối thiểu khóa GD chuẩn hóa, Agent ID hưởng chỉ tiêu, ngày nghiệp vụ, qualification/trạng thái và revision/hủy. Chưa resolve sale thì giữ unresolved, chưa cộng chỉ tiêu. Chi tiết hợp đồng ở [TDD, mục 4.4](BDSKD-9533-thiet-ke-ky-thuat.md#44-nhận-gd-từ-pipeline-qua-kafka).

## 5. Room, thông báo và truy nguyên

### 5.1. Room — loại sale thử thách khỏi số sale cấp hạn mức

| Bảng hiện có | Cột/khóa liên quan | Vai trò |
| --- | --- | --- |
| `user_registered_scope` | `username`, `registration_type`, `scope_type`, `scope_id`, `status` | Sale đăng ký dự án; `username = agency_cobroker.agent_profile_id` |
| `agency_cobroker`, `cobroker_profiles` | Link ACTIVE, `'SALE_MEMBER' = ANY(role_type)`, profile status theo `SaleActiveStatusPolicy` | Predicate sale hợp lệ hiện hành; khác tập sale cần báo cáo chu kỳ |
| `sale_batches` | `id`, `projects` (`jsonb`), `rules` (`jsonb`), `room_scope`, `room_mode` | Dự án đợt bán và luật room; `rules` không phải chính sách chu kỳ 9533 |
| `sale_batch_agencies` | `batch_id`, `agency_profile_id`, `status` | Đại lý tham gia đợt bán |
| `sale_batch_agency_rooms` | `id`, `batch_id`, `agency_profile_id`, `project_id`, `room_base`, `refill_granted`, `room_reserved` | Quỹ room; `project_id` NULL là quỹ SHARED theo code |
| `sale_batch_agency_rooms` | `exchange_quota_manual`, `advance_quota_manual`, `manual_input_at`, `manual_input_by` | Room nhập tay; tác động 9533 còn cần PO chốt |
| `room_ledger` | `pool_id` → room `id`; `batch_id`, `agency_profile_id`, `entry_type`, `delta`, `idempotency_key` | Ghi biến động room; không phải lịch sử chu kỳ sale |
| `sale_batch_units` | `batch_id`, `allocated_to_agency_profile_id`, `sold_by_agency_profile_id`, `allocation_status`, `sale_status` | Đối soát căn đang giữ/đã bán và tác động room sau recalc |

Room đang đếm DISTINCT `user_registered_scope.username` theo dự án đợt bán. Thêm điều kiện hồ sơ chu kỳ không bị `room_excluded` vào **query room riêng** sau khi có kết quả chu kỳ đã publish. Chưa có cột exclusion trong nguồn hiện tại. Không sửa shared `ACTIVE_LINK_JOIN` để tránh đổi các báo cáo/điểm dùng chung. Không đổi link thành INACTIVE để mô phỏng thử thách.

Index hiện có `uq_sba_rooms_batch_agency_project` UNIQUE `(batch_id, agency_profile_id, project_id) NULLS NOT DISTINCT`, chỉ áp dụng row status khác DELETED. Khi join room phải giữ chiều batch/project, không chỉ join theo đại lý; join với history/ledger cần aggregate để tránh nhân dòng.

### 5.2. Thông báo — US-05

`notification_outbox` có `object_type`, `object_id`, `type`, `scheduled_at` (`timestamptz`), `payload` (`jsonb`). UNIQUE `(object_type, object_id, type)` chỉ chống trùng row đang nằm trong outbox, không tự bảo đảm lịch sử gửi của nhiều kỳ/mốc nhắc.

Convention service/scheduler hiện tại xóa row khi đã xử lý gửi. Receiver còn tùy handler: handler phân phối hiện có thể nhận ID đại lý; 9533 gửi cho **sale qua tài khoản Agent**, cần handler/payload riêng, không copy nguyên receiver đại lý.

Đích đề xuất: `sale_cycle_notification_delivery` lưu lịch gửi/trạng thái đã gửi theo kỳ/mốc; dùng lại `notification_outbox` để giao việc gửi. Cấu hình lịch lấy từ policy chu kỳ mới, không lấy `distribution_reminder_log` làm lịch sử nhắc chu kỳ.

### 5.3. Audit/import/export

`audit_logs` có `organization_id`, `agency_profile_id`, `entity_type`, `entity_id`, `actor`, `action`, `field`, `value_before`, `value_after`, `changed_at`. `cobroker_profile_history` và `identity_verification_history` có before/after và thời điểm nghiệp vụ tương ứng. Đây là nền truy nguyên hiện hữu; chưa có event/handler/audit chuyên cho chu kỳ đã được xác minh.

Thiết kế đã review: dùng lại audit_logs để lưu thay đổi và log xuất; async_job để evaluate/rebuild/room; import_job/import_job_item để nhập ngày cũ. Reuse có điều kiện: thêm next_attempt_at/retry FAILED cho async_job và APPLIED/applied_at cho import_job_item. audit_logs đủ cấu trúc, bổ sung action/query. Review chi tiết tại mục 3.3 TDD; không coi tính năng 9533 đã hoàn tất chỉ vì có bảng chung. Excel dùng cùng nguồn/filter/scope với báo cáo, không có bảng dữ liệu Excel riêng cần đồng bộ.

## 6. Mapping phần cần bổ sung theo sáu US

Bảng sale_cycle_* là đề xuất mới; async_job và audit_logs là bảng có sẵn được tái sử dụng theo [TDD, mục 3](BDSKD-9533-thiet-ke-ky-thuat.md#3-cần-lưu-những-bảng-nào).

| US / chức năng | Bảng đề xuất | Dữ liệu chính | Nguồn hiện có nối vào |
| --- | --- | --- | --- |
| Nền danh sách / US-01, US-02 | `sale_cycle_profiles` | `agent_profile_id`, audience/org, `start_date`, monitoring, `cycle_classification`, `current_cycle_id`, `latest_official_cycle_id`, `latest_challenge_cycle_id`, `room_excluded` | Profile/link/đại lý; CMS cho O2O/Tự doanh |
| Chu kỳ / US-01 | `sale_cycle_period` | Hồ sơ chu kỳ, kind, policy, ngày đầu/cuối, target, `transaction_count`, achieved/result/lifecycle/generation | Engine tính từ hồ sơ chu kỳ + policy + facts |
| GD cá nhân / US-01 | `sale_cycle_transactions` | Khóa GD nguồn chuẩn hóa, sale/hồ sơ chu kỳ, ngày, qualification, revision | vhm-sale-pipeline → Kafka consumer; chốt payload/ID sale/GD |
| Cấu hình / US-04 | `sale_cycle_policy`, `sale_cycle_policy_rule` | Version/ngày hiệu lực/cơ chế; audience, tháng/chỉ tiêu chính thức và thử thách; lịch nhắc theo thiết kế | Người có quyền/nhập chính sách lịch sử; không dùng scoring config |
| Ngày bắt đầu / US-06 | Các field start date trên `sale_cycle_profiles`; `audit_logs` | Ngày, nguồn, override; before/after | Xác thực lần đầu đã được xác nhận, CMS, file import Chính sách, sửa thủ công có quyền |
| Thông báo / US-05 | `sale_cycle_notification_delivery` | Kỳ/mốc gửi/dedupe/trạng thái/lịch sử | Policy + period → `notification_outbox` |
| Excel / US-03 | `audit_logs` | Lần xuất, người xuất, filter/số dòng/tham chiếu file theo thiết kế | Query hồ sơ chu kỳ/period/profile giống báo cáo; tối đa 50.000 dòng theo SRS |
| Tính lại/chuyển kỳ/room | `async_job` | Công việc evaluate/rebuild/recalc và retry/dedupe | Ghi cùng transaction thay đầu vào/kết quả; worker cập nhật room hiện có |
| Truy nguyên thay đổi | `audit_logs` | Hành động, actor, before/after, thời điểm | Các service ghi ngày/policy/fact/kết quả |

Không dùng `agency_scoring_configs`, `sale_batch_scoring_configs` hoặc `project_assignment_policy` làm chính sách chu kỳ chỉ vì có tên config/policy. Chúng phục vụ điểm/phân dự án, chưa có mô hình tháng/chỉ tiêu chính thức/thử thách của 9533.

## 7. SQL đối soát và các điểm còn thiếu

File [BDSKD-9533-mapping-bang-db.sql](BDSKD-9533-mapping-bang-db.sql) chứa query metadata và thống kê không xuất dữ liệu cá nhân, chỉ dùng bảng hiện có. Chạy trong transaction read-only; kết quả thay đổi theo dữ liệu staging. Các query đối soát trong file đã được thực thi qua truy vấn tương đương trong phiên kiểm tra này; không có query đọc `sale_cycle_*` khi chưa migration.

Các phần chưa thể hoàn tất mapping từ DB này:

1. **Ngày bắt đầu bán:** nguồn xác thực nào thực sự đánh dấu lần đầu đủ điều kiện; ngày cũ theo file Chính sách; timezone timestamp cũ. HĐLĐ/created_at không thay nguồn này.
2. **GD cá nhân:** topic/payload Kafka của vhm-sale-pipeline, khóa nối sale nguồn ↔ Agent, ngày nghiệp vụ, khóa chống trùng, hủy/revision và replay dữ liệu lịch sử. Pipeline xác định GD đủ điều kiện; core lưu kết quả ghi nhận.
3. **O2O/Tự doanh và quyền vùng:** CMS/profile cung cấp org chart, employee code, audience và ngày nguồn; region/team ID trong core chưa đủ.
4. **Luật thay kết quả:** đạt giữa kỳ, policy tức thì/tuần tự, GD hồi tố, chuyển đại lý và room đã cấp/manual room cần PO xác nhận như mục 7 PRD.
5. **Chất lượng nguồn:** xử lý 2 link orphan profile trước backfill; dữ liệu thiếu phải có lý do, không tự phân loại sale không đạt.

Tham chiếu code từ gốc repo:

- `src/main/java/vn/vinhomes/cobroker/core/model/CoBrokerProfile.java`, `AgencyCobroker.java`, `AgencyProfile.java` và `model/jsonb/AgencyCobrokerEmploymentJsonb.java`.
- `src/main/java/vn/vinhomes/cobroker/core/service/applicant/IdentityVerificationConcludeService.java`, `ApplicantEkycCompletionService.java` và `entity/IdentityVerificationHistoryEntity.java`.
- `src/main/java/vn/vinhomes/cobroker/core/dto/distribution/event/SaleOrderChangeEvent.java`, `processor/PropertySoldProcessor.java`, `PropertySoldRecorder.java`.
- `src/main/java/vn/vinhomes/cobroker/core/repository/UserRegisteredScopeRepository.java`, `service/distribution/impl/RoomServiceImpl.java`, `entity/distribution/SaleBatchAgencyRoomEntity.java`.
- `src/main/java/vn/vinhomes/cobroker/core/entity/NotificationOutboxEntity.java`, `service/notification/outbox/NotificationOutboxService.java`.
