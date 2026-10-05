# BDSKD-9531 — Đối chiếu nguồn dữ liệu và DB xếp hạng riêng

Cập nhật 05/10/2026. Nguồn hồ sơ là các bảng profile-mw, datasource chỉ đọc; nguồn GD là Kafka sale-pipeline theo định hướng bộ 9533. Ledger 9531 phải có milestone/doanh số phù hợp, chưa có payload thực tế được cung cấp. Tất cả bảng sale_ranking_* là thiết kế mới trong DB riêng, chưa tạo.

Không dùng bảng/core-broker. Không thêm nguồn SAP trực tiếp: SAP là nguồn nghiệp vụ được PRD nêu, pipeline là điểm tích hợp dự kiến. Không suy rằng có tên cột nguồn là đã kiểm chứng đầy đủ giá trị hoặc mapping.

## 1. Những gì đã được đọc

| Nguồn | Đã đọc | Chưa đọc / chưa xác minh |
| --- | --- | --- |
| SRS 9531 local | US-01–06 và các bảng đặc tả | Phiên bản live Confluence; connector yêu cầu kết nối lại |
| PRD 9531 ODT | Bản 0.1 ngày 23/09/2026 | Q&A Excel, prototype và template ngoài file |
| Schema vhm-profile | Dùng khảo sát metadata 24 bảng/231 cột trong 9533 | Row nghiệp vụ, code profile-mw, enum và khóa người xuyên tài khoản |
| Pipeline Kafka | Định hướng tích hợp từ bộ 9533 | Contract cho 9531: HĐMB/VBCN, net revenue, attribution/revision/history |

## 2. Mapping profile-mw → bản đọc service mới

| Yêu cầu SRS | Bảng/field vật lý đã thấy | Field đích | Mapping cần đặc tả |
| --- | --- | --- | --- |
| ID tài khoản sale | user.id, user.username | accounts.external_user_id, agent_profile_id | username có đồng nhất Agent ID/pipeline sale ID không |
| Người giữ tích lũy khi đổi đại lý | user.id/employee_id/cobroker_profile_id là các ứng viên, chưa xác nhận semantics | profiles.stable_source_identity + accounts.profile_id | Khóa người hoặc alias mapping nguồn duyệt; không nối theo tên/phone |
| Họ tên | user.full_name/full_name_slug generated | accounts.display_name/name_search | Format/search collation và snapshot result/certificate |
| Mã nhân viên | user.employee_id, UNIQUE nullable | accounts.employee_code | Tài khoản có field; không tự dùng nó làm khóa người mọi audience |
| Mã định danh sale đại lý | user.properties và property alias có thể định nghĩa field | accounts.identity_code | Cần key cụ thể; chưa chứng minh cobroker_profile_id là CCCD |
| Điện thoại/email | user.phone/personal_email/work_email | accounts.phone/email | Chọn field theo nhóm và ưu tiên được duyệt |
| Trạng thái hoạt động | user.status | accounts.account_status | Enum Nháp/Chờ xác thực/Đang/Ngưng không suy từ default 1 |
| Role | user_roles.username/role_id → user.username/role.role_id | Adapter xác định audience/scope | Mã role 11/100/64/60/23/20/21 và status theo SRS |
| Bộ phận/vùng | user.team/region/department; team.id/parent_group/type | accounts.organization_id; organizations | Resolve varchar ID/mã; cấp tổ chức; vùng chủ quản đại lý |
| Đại lý hiện tại | user.cobroker_profile_id; team.tax_code/properties là dấu vết nguồn | accounts.agency_external_id, tên tổ chức nguồn | Không mặc định team hoặc cobroker_profile_id chính là đại lý; đặc tả rule mapping |
| Người quản lý/phạm vi | team.owner/parent_group; permission_setting | Scope tin cậy cho API | owner không tự cấp toàn quyền; priority allow/deny cần xác nhận |
| Danh mục dự án tùy chọn | team_distribute_setting.project_ids, không có bảng project trong 24 bảng | Project filter options nếu bật | JSON ID chưa có tên; chốt nguồn CMS danh mục hoặc deferred filter |

user.hire_date không là đầu vào tính điểm 9531. user.sale_score JSON và sales_member_tier hiện hữu không tự là bảng kết quả Kim cương/Bạch kim/Vàng theo quý; service mới không ghi vào đó.

## 3. Contract GD riêng cho 9531

| Thông tin | Dùng để | Thiếu thì sao? |
| --- | --- | --- |
| Tenant, nguồn và ID GD ổn định | Chống trùng và phân vùng dữ liệu | Event invalid/DLT, không đếm theo eventId |
| Sale ID + mapping tài khoản/người | Attribution doanh số đúng người, giữ qua chuyển đại lý | Lưu unresolved, chờ mapping |
| Đại lý tại thời điểm GD | Giữ attribution lịch sử | Không lấy đại lý hiện tại để ghi đè |
| Mua sơ cấp + milestone KH xác nhận HĐMB/VBCN | Đúng điều kiện SRS 9531 | Không dùng TTĐC/TTKQ của 9533 thay điều kiện |
| Timestamp milestone/ngày phân quý | GD thuộc Q nào | Không dùng received_at |
| Giá trị không VAT/KPBT và currency | Doanh số và điểm weighted | Không dùng giá tổng không rõ thuế/phí; không tự chuyển tiền tệ |
| Project ID | Số liệu theo dự án nếu filter bật | Không tự thêm filter không có dữ liệu |
| source_revision, eventId/time | Idempotency/order/audit | Cần hợp đồng revision thật, offset không là revision nghiệp vụ |
| business_status và correction type | Cancel vẫn tính; sửa fact riêng có kiểm soát | Không đồng nhất cancel và revoke ghi nhận |
| Completeness/watermark/replay | Chốt quý đúng với T−1 và dữ liệu lịch sử | Chờ dữ liệu; không phát hạng từ lịch sử thiếu |

Nếu pipeline chỉ gửi sale có 1 GD mà không có giá trị/mốc chuẩn thì chưa tính được doanh số 9531; cần mở rộng contract tại cùng nguồn pipeline, không chuyển qua bảng căn core-broker. HĐMB và VBCN của cùng GD phải dùng khóa chuẩn hóa, tránh hai lần ghi nhận.

## 4. Khác biệt dữ liệu 9531 và 9533

| Chủ đề | 9531 xếp hạng | 9533 theo dõi chu kỳ |
| --- | --- | --- |
| Kỳ | Quý dương lịch Q và Q−1 | Chính thức/thử thách theo ngày bán và số tháng |
| Điều kiện GD | KH xác nhận HĐMB/VBCN | GD đủ điều kiện TTĐC/TTKQ theo nguồn đã chốt |
| Số tiền | Cần net revenue không VAT/KPBT | Count GD là đầu vào chính |
| Hủy sau ghi nhận | Vẫn tính theo SRS | Thu hồi có thể cần rebuild theo luật được chốt |
| Định danh tích lũy | Giữ qua chuyển đại lý; cần người ổn định | Thiết kế 9533 hiện theo Agent ID, nối lịch sử còn chờ PO |
| Đạt kết quả | Tier chịu ngưỡng + quota/ties toàn pool | Chỉ tiêu theo kỳ của từng sale |
| Đầu ra | Báo cáo/badge/certificate | Báo cáo/room exclusion/thông báo |

Có thể dùng lại quy ước adapter/consumer/task ở mức thiết kế. Không dùng cùng một flag qualified/revoked hoặc cùng bảng result để gộp hai luật khác nhau.

## 5. Mapping US → DB riêng

| US | Bảng chính | Đầu ra |
| --- | --- | --- |
| US-01 | profiles/accounts/org + quarters/runs/results + transactions | Chỉ số/score tạm tính, hạng sau chốt |
| US-02 | Bản đọc account/org và results publish | Search/sort/filter/page20 đúng scope |
| US-03 | Cùng query báo cáo + audit_logs | Excel ≤50.000 rows, vượt tự cắt |
| US-04 | policy/rule + audit_logs + quarter/task | Cấu hình current/future, history, bootstrap quý cũ riêng |
| US-05 | Quarter final pointer + result của owner | Badge quý trước; không thêm bảng badge |
| US-06 | certificates + result snapshot | PNG/JPEG theo template/owner; queue file giữ riêng |

Tên bảng đầy đủ dùng prefix sale_ranking_; audit/task/certificate là bảng mới trong DB riêng. Không dùng import_job/audit_logs/notification_outbox có sẵn ở một service khác.

## 6. Đối soát trước nghiệm thu

1. Profile account role join không trùng; người qua hai Agent ID đúng mapping, org/scope đúng, inactive không mất.
2. Đối soát GD milestone/net revenue/date/attribution; confirm → cancel vẫn giữ; correction riêng có audit.
3. Nguồn đủ Q2/Q3/2026; GD ngoài quý không cộng nhầm; score dùng raw revenue Q−1/Q, không score Q−1.
4. Chốt pool/quota/min_score/ties với BO; lọc dự án/agency không đổi hạng đã tính.
5. Quarter final/run version và certificate thống nhất; input mới không publish run stale; thiếu nguồn chưa tier NONE.
6. Quyền 21 xem bộ phận nhưng không export; owner mới tải file; export >50.000 cắt đúng thứ tự.

Query nguồn dùng datasource chỉ đọc; query kết quả dùng DB service mới. Đợt tài liệu này không chạy SQL nghiệp vụ hoặc reconnect DB. [Luồng DB](BDSKD-9531-xep-hang-sale-theo-doanh-so-luong-du-lieu-va-lo-trinh.md) là tài liệu chính cho thứ tự ghi transaction.
