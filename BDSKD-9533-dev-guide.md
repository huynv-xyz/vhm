# BDSKD-9533 — Phân tích yêu cầu theo dõi kết quả bán hàng

> Tài liệu phân tích yêu cầu để PO, dev và AI agent cùng hiểu nghiệp vụ trước khi thiết kế và triển khai.
> Nguồn chính: PRD phiên bản 0.1, ngày 23/09/2026, đang chờ phê duyệt.
> Đối chiếu: code staging tại commit `31444ce6` và schema DB staging `cobroker_db`, kiểm tra ngày 05/10/2026.

## 1. PO đang muốn làm gì?

PO muốn biết từng sale có bán hàng hiệu quả trong khoảng thời gian được giao hay không. Kết quả này dùng để nhắc nhở sale, hỗ trợ quản lý đôn đốc và xác định sale còn được tính vào hạn mức căn của đại lý.

Mỗi sale được theo dõi qua hai giai đoạn:

- **Bán chính thức:** sale có thời gian để đạt chỉ tiêu giao dịch. Nếu đạt, sale tiếp tục một chu kỳ bán chính thức mới.
- **Cảnh báo:** sale chưa đạt trong giai đoạn chính thức được thêm thời gian để bán, nhưng không được tính vào room. Nếu bán được thì trở lại chính thức; nếu hết thời gian vẫn không bán được thì thuộc diện đề xuất ngừng hợp tác.

Phạm vi áp dụng gồm sale Đại lý, O2O và Tự doanh. Mỗi nhóm có thể có chính sách khác nhau.

Đây là chức năng đánh giá hiệu quả bán hàng theo chu kỳ. Nó cần dữ liệu giao dịch của **từng sale**, không chỉ dữ liệu một đại lý đã bán bao nhiêu căn.

## 2. Một ví dụ để hiểu toàn bộ luồng

Giả sử sale đại lý bắt đầu bán ngày 01/01/2026. Chính sách là 4 tháng bán chính thức, 6 tháng cảnh báo và cần 1 giao dịch.

| Thời điểm | Tình huống | Kết quả theo hướng mô tả trong PRD |
| --- | --- | --- |
| 01/01–30/04 | Sale đang bán chính thức | Theo dõi số GD và thời gian còn lại |
| 15/03 | Sale đạt 1 GD | Đạt yêu cầu; tiếp tục chu kỳ đến hết 30/04 |
| 01/05 | Chu kỳ trước đã đạt | Mở chu kỳ bán chính thức tiếp theo |
| 01/05, nếu chu kỳ trước không đạt | Sale chưa có GD | Chuyển sang cảnh báo, không tính vào room |
| 10/06, trong cảnh báo | Sale có GD đủ điều kiện | Kết thúc cảnh báo; ngày 11/06 bắt đầu chính thức mới |
| Hết cảnh báo vẫn không đạt | Sale không đáp ứng chính sách | Đưa vào diện đề xuất ngừng hợp tác |

Ví dụ dùng các tháng có ngày bắt đầu là ngày 1 để tránh giả định về ngày cuối tháng. PRD chưa quy định đầy đủ cách tính ngày kết thúc.

**Hai quyết định chưa được chốt:** đạt giữa kỳ chính thức có thực sự giữ đến cuối kỳ không; trong cảnh báo chỉ cần 1 GD hay phải đủ chỉ tiêu cấu hình. Không lấy ví dụ này thay cho quyết định của BO.

## 3. Người dùng cần nhìn thấy và làm được gì?

### Khối Chính sách / Kinh doanh

Xem toàn bộ sale, biết ai đạt, ai chưa đạt, ai đang cảnh báo và ai thuộc diện đề xuất ngừng hợp tác. Có thể lọc và xuất Excel để làm việc với các bộ phận liên quan.

Thiết lập chính sách theo Đại lý, O2O, Tự doanh: thời gian chính thức, thời gian cảnh báo, số GD và ngày hiệu lực. Một nhóm đối tượng không được nằm trong hai khối của cùng cấu hình.

### Admin đại lý / Quản lý kinh doanh

Xem sale thuộc bộ phận mình quản lý. Dùng số ngày còn lại và số GD để đôn đốc trước khi sale bị chuyển cảnh báo.

PRD chưa xác định role kỹ thuật, cách xác định tổ chức quản lý và quyền riêng cho xuất báo cáo hoặc cấu hình.

### Sale

Nhận thông báo trên web/app Agent khi sắp hết chu kỳ và khi kết thúc chu kỳ. PRD chưa mô tả một màn hình riêng để sale xem tiến độ của mình.

## 4. Những việc nằm trong yêu cầu PRD

| Phần việc | Yêu cầu cần đáp ứng |
| --- | --- |
| Báo cáo | Hiển thị chu kỳ chính thức/cảnh báo, số ngày còn lại, số GD và phân loại từng sale |
| Tìm kiếm và lọc | Khoanh vùng sale theo nhóm/tổ chức, trạng thái hoạt động và phân loại |
| Xuất Excel | Xuất thông tin báo cáo phục vụ xử lý nghiệp vụ |
| Chính sách | Cấu hình theo nhóm, có ngày hiệu lực; xem xét cơ chế chuyển luật |
| Tính kết quả | Tự ghi nhận GD và chuyển giai đoạn theo chính sách |
| Nhắc nhở | Gửi thông báo trước hạn và khi kết thúc chu kỳ |
| Ngày bắt đầu bán | Ghi từ hoàn thành xác thực, có phương án nhập tay |
| Room | Loại sale cảnh báo khỏi điều kiện tính hạn mức căn; khôi phục khi trở lại chính thức |

PRD loại khỏi đợt này việc reset chu kỳ thủ công cho từng sale và màn hình toàn bộ lịch sử chu kỳ. Không vì vậy mà kết luận hệ thống không cần dữ liệu lịch sử để kiểm chứng kết quả.

## 5. Đối chiếu với hệ thống hiện tại

Đã kiểm tra schema bằng kết nối chỉ đọc, không sửa dữ liệu và không đưa thông tin kết nối vào tài liệu. Các kết luận về DB dưới đây chỉ giới hạn trong schema `cobroker_db` mà tài khoản được cung cấp nhìn thấy. Chưa kiểm tra DB của CMS, SAP hay Housing.

### 5.1. Đã có dữ liệu sale và quan hệ với đại lý

Hệ thống hiện có:

- `cobroker_profiles`: hồ sơ con người, thông tin cá nhân, account và trạng thái profile.
- `agency_cobroker`: quan hệ làm việc giữa người đó với đại lý, vai trò, trạng thái quan hệ và tài khoản Agent `agent_profile_id`.
- `agency_profiles`: thông tin đại lý, team và vùng quản lý.
- `user_registered_scope`: phạm vi/dự án sale đăng ký.

Điểm cần hiểu: **một người và tài khoản sale tại đại lý là hai khái niệm khác nhau**. Code hiện tại mô tả chuyển đại lý có thể tạo agent-profile mới, trong khi hồ sơ con người vẫn được giữ.

Vì vậy PO phải xác định chu kỳ theo người hay theo tài khoản tại đại lý. Nếu theo người, chuyển đại lý có thể giữ kết quả cũ. Nếu theo tài khoản, người đó có thể bắt đầu lại. PRD chưa đưa ra quyết định này.

Tại thời điểm kiểm tra, DB staging có 1.004 quan hệ mang vai trò `SALE_MEMBER`; tất cả các quan hệ này có `agent_profile_id`. Con số này là số quan hệ, không phải kết luận có 1.004 người thuộc phạm vi tính chu kỳ. Cũng không đại diện số liệu production.

**Đánh giá:** có nền dữ liệu cho sale đại lý. Chưa xác minh nguồn danh sách và tổ chức của O2O/Tự doanh, nên chưa thể kết luận cobroker-core đã có đủ toàn bộ đối tượng PRD.

### 5.2. Có dữ liệu xác thực, nhưng chưa có ngày bắt đầu bán riêng

Trong schema hiện tại, `cobroker_profiles` và `agency_cobroker` chưa có cột riêng cho ngày bắt đầu bán.

Hồ sơ đăng ký `cobroker_applicant_agencies_submission` có trạng thái, `approve_at`, thông tin eKYC và tiến trình xác thực. Bảng `identity_verification_history` có thời điểm sự kiện và các sự kiện OCR/eKYC, duyệt thủ công, từ chối và xử lý lại.

**Điều này chưa có nghĩa lấy `approve_at`, ngày tạo hồ sơ hoặc một sự kiện eKYC bất kỳ là đúng ngày bắt đầu bán.** Hoàn thành xác thực có thể gồm nhiều bước và nhiều lần xử lý. PO cần chốt mốc nghiệp vụ trước khi dùng lịch sử để suy ra ngày cho sale cũ.

Cần xác nhận:

- Hoàn thành OCR và eKYC nghĩa là cả hai đã đạt hay một bước cụ thể?
- Duyệt thủ công có được coi là hoàn thành không?
- Xác thực lại có thay đổi ngày bắt đầu bán không?
- O2O/Tự doanh lấy ngày nào khi không đi cùng luồng onboarding đại lý?

**Đánh giá:** có dữ liệu để điều tra và đối soát ngày xác thực; chưa đủ căn cứ để tự động suy ra ngày bắt đầu bán cho toàn bộ sale.

### 5.3. Có luồng nhận giao dịch bán căn, nhưng chưa đủ để tính KPI từng sale

Code hiện tại có `PropertySoldConsumer` nhận sự kiện sale order từ Housing. `PropertySoldProcessor` chuyển trạng thái căn dựa trên mã giao dịch/trạng thái SAP.

Payload được code hiện tại sử dụng có loại giao dịch, trạng thái, mã số thuế đại lý và thời gian ký cọc/HĐMB. Không có field định danh sale trong phần payload đã được ánh xạ vào `SaleOrderChangeEvent`.

DB có `sale_batch_units` với trạng thái bán, ngày ký/bán và đại lý bán căn. Trong các cột đã kiểm tra, không có cột định danh sale thực hiện giao dịch. Bảng này quản lý căn trong đợt bán, không phải sổ giao dịch KPI cá nhân đầy đủ.

Tại thời điểm kiểm tra có 243 bản ghi căn mang trạng thái `SOLD`. Không được dùng con số này làm số GD hợp lệ của 9533: chưa biết mốc SAP có khớp PRD không, sale nào được hưởng GD và một căn xuất hiện trong các đợt có gây đếm trùng hay không.

PRD cũng chưa nhất quán về mốc GD: luồng mô tả TTKQ/ĐC, trong khi phần hệ thống liên quan nói thời điểm ký VBCN/HĐMB.

**Đánh giá:** có đường tích hợp trạng thái bán căn, nhưng chưa chứng minh đủ dữ liệu tính GD của từng sale. Cần xác nhận nguồn giao dịch cá nhân và quy tắc ghi nhận; không lấy trạng thái `SOLD` hiện tại thay thế ngay cho KPI.

### 5.4. Có room tính theo lực lượng sale, nên yêu cầu sẽ ảnh hưởng vận hành thật

Room hiện tại có phần tính theo số sale hợp lệ của đại lý trong các dự án thuộc đợt bán. Code kiểm tra quan hệ đại lý đang hoạt động, role `SALE_MEMBER`, trạng thái profile theo chính sách hiện hành và đăng ký dự án đang hoạt động. Sale có nhiều dự án trong cùng đợt được đếm một lần.

Điều kiện này chưa có kiểm tra chu kỳ cảnh báo.

Nếu 9533 loại một sale khỏi số lượng được tính, room đại lý có thể giảm. Tuy nhiên PRD chưa nói rõ phải làm gì nếu đại lý đã sử dụng room hoặc có căn đang phân bổ.

Cần tách ba quyết định:

1. Sale cảnh báo bị loại khỏi công thức tính room đại lý hay không?
2. Sale đó có bị chặn trực tiếp thao tác lấy căn hay không?
3. Những căn/room đã cấp có bị thu hồi hay được giữ đến khi xử lý xong?

**Đánh giá:** đây là thay đổi điều kiện cấp nguồn lực, cần phối hợp nghiệp vụ Giỏ hàng. Không thể coi là chỉ thêm một nhãn trên báo cáo. Cũng không nên đổi trạng thái tài khoản thành inactive để mô phỏng cảnh báo.

### 5.5. Chưa thấy dữ liệu theo dõi chu kỳ trong schema đã kiểm tra

Danh sách bảng trong `cobroker_db` chưa có bảng chuyên theo dõi chu kỳ bán/cảnh báo và chính sách chu kỳ của từng sale.

Hệ thống có các cấu hình điểm đại lý và đợt bán, nhưng đó là cấu hình phục vụ nghiệp vụ khác. Chưa có căn cứ coi chúng là chính sách chu kỳ của PRD.

**Đánh giá:** phần theo dõi chu kỳ là năng lực cần bổ sung. Chưa cần quyết định schema hoặc API ở bước phân tích này; trước hết cần chốt luật và dữ liệu đầu vào.

### 5.6. Có hạ tầng thông báo và lịch sử để tham khảo

Hệ thống có `notification_outbox` và dịch vụ lập lịch gửi/hủy thông báo. Đây là nền kỹ thuật có thể sử dụng sau khi rõ yêu cầu.

PRD vẫn cần xác định gửi khi nào, gửi cho ai và trường hợp nào không gửi. Có hạ tầng gửi không đồng nghĩa đã có đầy đủ nghiệp vụ thông báo 9533.

## 6. Những quyết định nghiệp vụ cần PO/BO trả lời

### 6.1. Giao dịch nào được tính cho sale?

Đây là câu hỏi quan trọng nhất vì mọi phân loại đều dựa vào số GD.

Cần thống nhất mốc SAP được tính, khóa duy nhất của GD, ngày nghiệp vụ và định danh sale. Đồng thời xác định quản lý được tính GD cá nhân hay GD của đội; GD chuyển sale, hủy hoặc đồng bộ trễ có làm thay đổi kết quả cũ không.

Ví dụ cần trả lời: GD xác nhận ngày 30/04 nhưng hệ thống nhận ngày 02/05 có được dùng để cứu chu kỳ chính thức kết thúc 30/04 không? Nếu có, kết quả cảnh báo và room đã thay đổi phải xử lý thế nào?

### 6.2. Khi nào mở chu kỳ mới?

PRD có hướng giữ nguyên kỳ chính thức khi sale đạt giữa kỳ, nhưng còn chờ BO xác nhận.

Trong cảnh báo, tài liệu nói có GD thì trở lại chính thức. Nếu chỉ tiêu cấu hình là 2 mà sale mới có 1 GD, chưa rõ phải trở lại chính thức hay tiếp tục cảnh báo.

Cũng cần chốt cách tính tháng, ngày kết thúc và thời điểm chốt GD cuối kỳ, đặc biệt với ngày bắt đầu 29/30/31.

### 6.3. Kết thúc cảnh báo có hành động gì?

PRD có chỗ nói “đề xuất chấm dứt”, có chỗ nói “chấm dứt vĩnh viễn”. Hai cách diễn đạt dẫn đến hành vi khác nhau.

Cần xác nhận đây chỉ là danh sách để BO quyết định hay hệ thống tự khóa tài khoản/chấm dứt hợp tác. Nếu là đề xuất, ai xử lý tiếp? Nếu có GD đến muộn hoặc sale quay lại hợp tác, trạng thái được phục hồi thế nào?

Hiện chưa đủ căn cứ yêu cầu hệ thống tự khóa account hoặc tự chấm dứt hợp đồng.

### 6.4. Chính sách mới áp dụng cho ai và từ lúc nào?

BR-03 yêu cầu hoàn thành chu kỳ đang chạy theo luật cũ rồi dùng luật mới.

Cần xác định “chu kỳ đang chạy” là giai đoạn chính thức hoặc cảnh báo, hay toàn bộ cặp hai giai đoạn. Ví dụ sale kết thúc chính thức không đạt sau ngày luật mới có hiệu lực: cảnh báo sắp mở dùng số tháng/chỉ tiêu cũ hay mới?

Cần chốt cả trường hợp đổi nhóm Đại lý/O2O/Tự doanh, nhiều chính sách tương lai và sửa/hủy cấu hình đã lưu.

### 6.5. Sale hiện có bắt đầu được theo dõi thế nào?

Hệ thống đã có sale đang hoạt động và dữ liệu xác thực nhiều thời điểm. PRD chưa nói khi go-live phải tính lại từ ngày bắt đầu bán cũ hay cho tất cả bắt đầu kỳ mới.

Nếu tính lại lịch sử, phải có đủ GD và chính sách tương ứng. Nếu thiếu ngày bắt đầu hoặc nguồn GD, không nên tự dùng ngày tạo profile hay tự kết luận sale không đạt.

PO cần xác định cách khởi tạo, người cung cấp dữ liệu và cách đối soát trước khi áp dụng tác động room.

## 7. Phạm vi và mốc bàn giao cần làm rõ

PRD dự kiến UAT 25–30/10/2026, go-live 31/10/2026. Riêng công cụ cấu hình động có thể tới 30/11/2026.

Nếu engine và báo cáo chạy trước UI cấu hình, phải xác định chính sách ban đầu được nhập bằng cách nào, ai phê duyệt và cách thay đổi chính sách trong giai đoạn đó.

Cần chốt rõ đợt tháng 10 có những phần nào: báo cáo, thông báo, tính chu kỳ, dữ liệu cũ và tác động room. Không mặc định tất cả đều bàn giao cùng ngày chỉ vì thuộc chung PRD.

## 8. Khi nào có thể coi yêu cầu đủ rõ để dev làm?

PO/BO và nhóm kỹ thuật cần thống nhất được các ví dụ đầu vào/kết quả sau:

| Tình huống | Kết quả phải xác định được |
| --- | --- |
| Sale đạt giữa kỳ chính thức | Giữ kỳ hay mở kỳ mới, ngày nào và GD tính vào đâu |
| Sale hết chính thức không đạt | Khi nào vào cảnh báo và room thay đổi cụ thể thế nào |
| Sale cảnh báo có GD nhưng chưa đủ chỉ tiêu | Tiếp tục cảnh báo hay trở lại chính thức |
| Sale hết cảnh báo không đạt | Gán nhãn hay hành động tự động; ai xử lý tiếp |
| GD đến trễ hoặc bị hủy | Có sửa kết quả cũ và hoàn tác room/thông báo không |
| Chính sách đổi giữa kỳ | Chính sách nào áp dụng ở giai đoạn kế tiếp |
| Sale chuyển đại lý/nhóm | Giữ hay bắt đầu lại chu kỳ, GD thuộc tài khoản nào |
| Sale thiếu ngày bắt đầu hoặc thiếu dữ liệu GD | Hiển thị và xử lý ra sao, không mặc định là không đạt |
| Sale có lịch sử trước go-live | Cách khởi tạo và đối soát kết quả |

Sau khi chốt các ví dụ này mới đặc tả chi tiết các cột báo cáo, quyền, nội dung thông báo và hợp đồng tích hợp. Đây là thứ tự phân tích, không phải yêu cầu dừng mọi công việc kỹ thuật.

## 9. Kết luận về yêu cầu trên hệ thống hiện tại

Hệ thống có nền dữ liệu và chức năng cho sale đại lý, xác thực, đăng ký dự án, room và gửi thông báo. Có thể dùng các phần đó để phát triển yêu cầu.

Ba khoảng trống cần giải quyết trước là **ngày bắt đầu bán đúng nghiệp vụ**, **giao dịch được ghi nhận cho từng sale** và **luật chuyển chu kỳ/tác động room**. Phạm vi O2O/Tự doanh cũng cần xác minh nguồn dữ liệu ngoài schema đã kiểm tra.

Chưa nên chốt phương án “lấy dữ liệu đang có rồi chạy job đếm ngày”: hiện trạng chưa chứng minh có đầy đủ dữ liệu KPI cá nhân, còn PRD chưa chốt các quy tắc quyết định kết quả.

## 10. Tham chiếu để kiểm chứng hiện trạng

Tài liệu phân tích PRD riêng: [BDSKD-9533-phan-tich-PRD.md](BDSKD-9533-phan-tich-PRD.md).

Các file code đã đối chiếu, đường dẫn tính từ gốc repo:

- Hồ sơ và quan hệ đại lý: `src/main/java/vn/vinhomes/cobroker/core/model/CoBrokerProfile.java`, `AgencyCobroker.java` cùng thư mục.
- Luồng xác thực: `src/main/java/vn/vinhomes/cobroker/core/service/applicant/ApplicantEkycCompletionService.java`.
- GD bán căn: `src/main/java/vn/vinhomes/cobroker/core/consumer/PropertySoldConsumer.java`, `src/main/java/vn/vinhomes/cobroker/core/processor/PropertySoldProcessor.java`, `src/main/java/vn/vinhomes/cobroker/core/dto/distribution/event/SaleOrderChangeEvent.java`.
- Điều kiện đếm sale: `src/main/java/vn/vinhomes/cobroker/core/repository/UserRegisteredScopeRepository.java`, phương thức `countValidSalesByAgency`.
- Room: `src/main/java/vn/vinhomes/cobroker/core/service/distribution/impl/RoomServiceImpl.java`, phương thức `computeRoom`.
- Thông báo: `src/main/java/vn/vinhomes/cobroker/core/service/notification/outbox/NotificationOutboxService.java`.

Schema và số lượng tổng hợp được đọc ngày 05/10/2026. Số liệu staging có thể thay đổi và không dùng để suy ra số liệu production. Tài liệu không chứa mật khẩu hay thông tin cá nhân của sale.
