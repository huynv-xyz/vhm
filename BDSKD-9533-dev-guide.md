# BDSKD-9533 — Phân tích yêu cầu theo dõi chu kỳ bán hàng

**Nguồn yêu cầu:** [SRS trên Confluence](https://vin3s.atlassian.net/wiki/spaces/BMAS/pages/3174532173), bản 7, cập nhật 01/10/2026; đọc lại ngày 05/10/2026, gồm US-01 đến US-06 và các ảnh thiết kế nhúng. Nội dung SRS đang **Under approve**; trạng thái trang `current` không có nghĩa yêu cầu đã được phê duyệt.

**Hiện trạng đối chiếu:** code `staging` tại commit `31444ce6` và schema DB staging `cobroker_db`, kiểm tra chỉ đọc ngày 05/10/2026. Chưa kiểm tra DB CMS, SAP và Housing.

Guide giải thích nghiệp vụ trước, sau đó đối chiếu hệ thống và chỉ ra các điểm cần xác nhận. Chưa chốt schema, API hoặc phương án triển khai. Những yêu cầu đã có trong SRS không được coi là câu hỏi còn mở; ghi chú chờ BO và các mâu thuẫn được nêu riêng ở mục 9.

## 1. Chức năng này giải quyết việc gì?

Hệ thống theo dõi từng sale có đạt số giao dịch tối thiểu trong thời gian quy định hay không. Kết quả dùng để quản lý đôn đốc, gửi nhắc nhở và xác định sale còn được tính để cấp hạn mức căn (room).

Sale trải qua hai loại chu kỳ:

- **Bán chính thức:** có thời gian đạt chỉ tiêu và duy trì điều kiện được tính vào room. Đạt thì tiếp tục chu kỳ chính thức mới.
- **Thử thách:** chưa đạt ở chính thức được thêm cơ hội bán, nhưng không được tính để cấp room. Đạt chỉ tiêu thì trở lại chính thức; hết thử thách không đạt thì đề xuất chấm dứt hợp tác.

“Chu kỳ cảnh báo” trong PRD và một số ảnh SRS là cách gọi khác của “Chu kỳ thử thách”. Guide dùng tên **Chu kỳ thử thách** theo phần nguyên tắc SRS.

**Trạng thái hoạt động tài khoản và phân loại chu kỳ là hai thông tin riêng.** SRS yêu cầu chu kỳ vẫn chạy dù trạng thái hoạt động sale thay đổi. Không dùng trạng thái ngưng hoạt động để thay cho thử thách.

## 2. Sale nào được theo dõi, ai được xem?

### Sale thuộc phạm vi — US-01

| Nhóm | Điều kiện SRS |
| --- | --- |
| Đại lý | Thuộc một đại lý cụ thể, có role Sale member (21) |
| O2O | Thuộc khối O2O, có role 21, 20, 23 hoặc 60 |
| Tự doanh | Thuộc khối Tự doanh, có role 21, 20, 23 hoặc 60 |

Role 20/23/60 tương ứng Trưởng nhóm/Giám đốc phòng/Giám đốc miền. Các cấp quản lý O2O/Tự doanh cũng được theo dõi, không chỉ sale member. Chưa rõ GD của họ là GD cá nhân hay GD đội.

SRS dẫn khối O2O `1060` và Tự doanh `1132` trên CMS staging. Phải xác minh mapping trước khi dùng ở môi trường khác.

### Phạm vi xem báo cáo — US-01/02

| Người dùng | Phạm vi xem |
| --- | --- |
| Admin hệ thống (11) / Quản lý đại lý (100) | Toàn bộ sale Vinhomes |
| Admin đại lý (64) | Sale thuộc đại lý mình quản lý |
| Quản lý vùng chủ quản đại lý | Sale thuộc các đại lý mình quản lý |
| Giám đốc miền (60) | Sale thuộc miền mình quản lý |
| Giám đốc phòng (23) | Sale thuộc phòng mình quản lý |
| Trưởng nhóm (20) | Sale thuộc nhóm mình quản lý |

SRS đã nêu phần lớn role và phạm vi. Những ghi chú còn mở là tái sử dụng role 100 hay thêm role mới, cách xác định vùng chủ quản và quyền export có cần `USER_EXPORTER (25)` không.

## 3. Ngày bắt đầu bán được ghi nhận thế nào? — US-06

Ngày này mở chu kỳ chính thức đầu tiên, không mặc định là ngày tạo account hoặc ngày thêm sale vào đại lý.

| Đối tượng | Ghi nhận và chỉnh sửa |
| --- | --- |
| Sale đại lý mới | Tự ghi ngày hoàn thành OCR/eKYC trên Agent, chuyển sang “Đã xác thực” |
| Sale đại lý | Admin hệ thống (11) / Quản lý đại lý (100) được sửa trên profile |
| Admin đại lý (64) | Chỉ được xem |
| O2O/Tự doanh | QL KD cung cấp để IT vận hành nhập trên CMS; cập nhật trên CMS profile |
| Tài khoản đã tồn tại | Chính sách cung cấp dữ liệu để IT import một lần |

Ảnh US-06 bổ sung trường ngày vào màn chỉnh sửa và màn chi tiết sale đại lý. Phần CMS dẫn ảnh “Ngày BC” nhưng vẫn ghi chú cần kiểm tra ý nghĩa và khả năng tái sử dụng. Chưa được coi field này là nguồn đã chốt.

**Khi ngày bắt đầu bán thay đổi, SRS yêu cầu tự tính lại chu kỳ.** Phạm vi hồi tố GD, room và thông báo đã gửi chưa được mô tả đầy đủ.

## 4. Chu kỳ chạy như thế nào? — US-01

### 4.1. Mốc thời gian

SRS đã quy định:

```text
Ngày bắt đầu chu kỳ đầu tiên = Ngày bắt đầu bán
Ngày kết thúc = Ngày bắt đầu + Số tháng cấu hình - 1 ngày
Ngày bắt đầu chu kỳ tiếp = Ngày kết thúc chu kỳ trước + 1 ngày
Số ngày còn lại = Ngày kết thúc - Ngày hiện tại
```

Ví dụ: chính thức 3 tháng bắt đầu 01/01/2026 kết thúc 31/03/2026. Ngày kết thúc vẫn thuộc chu kỳ; chuyển giai đoạn vào hôm sau. “Số ngày còn lại = 0” không có nghĩa chuyển thử thách ngay đầu ngày kết thúc.

Công thức đã có. Cách cộng tháng khi bắt đầu ngày 29/30/31, timezone và số ngày còn lại của chu kỳ đã qua cần thống nhất thêm.

### 4.2. Trong chu kỳ chính thức

- GD < chỉ tiêu: **Chu kỳ bán - Chưa đạt yêu cầu**.
- GD >= chỉ tiêu: **Chu kỳ bán - Đạt yêu cầu**.

Phần nguyên tắc SRS nói đạt giữa kỳ vẫn bán đến hết ngày kết thúc. Nếu cuối kỳ đạt, hôm sau mở chính thức mới.

**Lưu ý nguồn:** SRS vẫn còn ghi chú BO xác nhận reset ngay hay giữ đến hết kỳ. Guide mô tả theo phần nguyên tắc hiện có; cần giải quyết ghi chú để coi luật là cuối cùng.

### 4.3. Chuyển thử thách

Hết chính thức mà GD < chỉ tiêu thì hôm sau mở thử thách. Sale tiếp tục bán nhưng không được tính để cấp room.

### 4.4. Đạt trong thử thách

SRS quy định **đủ số GD yêu cầu** thì đóng thử thách trong ngày đạt. Hôm sau mở chính thức mới, đếm lại từ đầu và khôi phục điều kiện được tính để lấy room.

Không hiểu là bất kỳ 1 GD nào cũng đủ. Với chỉ tiêu 2, điều kiện đạt trong nguyên tắc là GD >= 2.

### 4.5. Hết thử thách không đạt

Bảng phân loại nói hết thử thách chưa đạt số GD tối thiểu thì **Đề xuất chấm dứt hợp tác**, không tự reset/sinh thêm chu kỳ.

Đoạn nguyên tắc 4 lại viết “vẫn không có GD”. Trường hợp có 1 GD nhưng chỉ tiêu 2 chưa được diễn đạt nhất quán; cần BO xác nhận thất bại khi GD < chỉ tiêu.

SRS mô tả nhãn đề xuất và thông báo thuộc diện xem xét. Không có yêu cầu tự khóa account hoặc tự chấm dứt hợp đồng.

### 4.6. Ví dụ hoàn chỉnh

Giả sử chính sách 4 tháng chính thức, 6 tháng thử thách, mỗi giai đoạn cần 1 GD; bắt đầu bán 01/01/2026.

| Tình huống | Kết quả theo phần nguyên tắc SRS |
| --- | --- |
| Chính thức đầu tiên | 01/01–30/04/2026 |
| Đạt ngày 15/03 | Gán Đạt yêu cầu; giữ kỳ đến 30/04 |
| Hết 30/04 đã đạt | Chính thức mới 01/05–31/08 |
| Hết 30/04 chưa đạt | Thử thách 01/05–31/10 |
| Đạt trong thử thách ngày 10/06 | Đóng thử thách 10/06; chính thức mới từ 11/06 |
| Hết 31/10 không đạt | Từ 01/11 đề xuất chấm dứt; không tự mở kỳ tiếp |

Ví dụ không phải cấu hình production đã duyệt. Luật đạt giữa kỳ và thử thách có GD nhưng chưa đủ vẫn có lưu ý nguồn nêu trên.

## 5. Báo cáo và thao tác cần có — US-01/02/03

### Thông tin hiển thị

Vào **Quản lý sale → Theo dõi chu kỳ bán hàng**.

| Nhóm | Các cột |
| --- | --- |
| Định danh | STT, họ tên, ID hệ thống Agent, mã nhân viên O2O/Tự doanh từ CMS, mã định danh đại lý từ Agent |
| Liên hệ/tổ chức | SĐT, email, bộ phận KD trực thuộc trực tiếp, vùng quản lý |
| Theo dõi | Trạng thái hoạt động, ngày bắt đầu bán, loại chu kỳ “x tháng - y tháng” |
| Chính thức gần nhất | Ngày bắt đầu, ngày kết thúc, số ngày còn lại, số GD |
| Thử thách gần nhất | Ngày bắt đầu, ngày kết thúc, số ngày còn lại, số GD |
| Kết quả | Ngày GD gần nhất, phân loại |

Trạng thái hoạt động gồm Đang hoạt động, Ngưng hoạt động, Chờ xác thực, Nháp. Cần mapping với trạng thái hiện tại.

### “Chu kỳ gần nhất” hiển thị thế nào?

| Sale đang ở đâu? | Nhóm chính thức | Nhóm thử thách |
| --- | --- | --- |
| Chính thức | Chính thức hiện tại | Dấu `-`, kể cả đã từng thử thách |
| Thử thách | Chính thức thất bại liền trước | Thử thách hiện tại |
| Đề xuất chấm dứt | Chính thức cuối cùng | Thử thách cuối cùng |

Đây là báo cáo hiện trạng, không phải màn hình toàn bộ lịch sử.

### Search/sort/filter và phân trang

- Pin STT/họ tên; mặc định họ tên A–Z; tối đa 20 dòng/trang, có chuyển trang.
- Search gần đúng theo họ tên, ID hệ thống, mã nhân viên, mã định danh.
- Sort họ tên, thời gian và số GD theo tăng/giảm; cần xác định cụ thể các cột thời gian.
- Bộ phận KD: chọn cấp Tất cả/Miền/Phòng/Đội nhóm, rồi chọn nhiều đơn vị. Mặc định theo cấp cao nhất user được xem.
- Trạng thái hoạt động và phân loại: chọn nhiều, mặc định tất cả.

SRS còn ghi chú bộ lọc tổ chức chưa hợp lý cho vùng chủ quản đại lý. Tổng quan US-02 nhắc filter thời gian/xếp hạng doanh số/GD, nhưng bảng chi tiết và ảnh không mô tả; cần xác nhận có phải nội dung thừa từ báo cáo xếp hạng.

### Excel

Xuất **toàn bộ kết quả đang lọc**, đầy đủ cột tương ứng báo cáo, không chỉ trang đang xem. Tối đa 50.000 dòng; SRS yêu cầu tự cắt khi vượt.

Ảnh US-03 có hộp xác nhận số sale cần xuất, Hủy bỏ/Xác nhận và thông báo lịch sử xuất sẽ được lưu. Đây là yêu cầu thể hiện trên thiết kế; chưa có mô tả nội dung lịch sử phải lưu hoặc màn xem lịch sử.

Template dẫn ra ngoài mang tên báo cáo xếp hạng. Chưa kiểm tra file Excel đó, nên chưa xác nhận có đúng các cột tính năng này.

## 6. Cấu hình cần cho phép gì? — US-04

Admin hệ thống (11) / Quản lý đại lý (100) vào **Cấu hình chu kỳ** để tạo cấu hình và xem lịch sử đã lưu.

| Nội dung | Yêu cầu SRS |
| --- | --- |
| Ngày hiệu lực | UI chỉ chọn tương lai, không chọn hôm nay/quá khứ |
| Cơ chế áp dụng | Chọn tức thì hoặc tuần tự |
| Đối tượng | Đại lý/O2O/Tự doanh; mỗi nhóm chỉ thuộc một box trong cấu hình |
| Loại chu kỳ | Cố định chính thức và thử thách, không thêm loại mới |
| Thời gian | Số tháng của chu kỳ |
| Chỉ tiêu | Số GD cần đạt |
| Nhắc trước hạn | Nhiều mốc ngày nhắc trước và giờ:phút |
| Báo kết thúc | Giờ:phút gửi ngày sau khi kết thúc |

**Tức thì:** reset tất cả sale về đầu chu kỳ khi cấu hình mới bắt đầu hiệu lực. Chưa rõ có bao gồm thử thách/đề xuất chấm dứt và giữ GD cũ không.

**Tuần tự:** sale bắt đầu bán sau ngày áp dụng dùng cấu hình mới; sale đã hoạt động hoàn thành kỳ hiện tại rồi dùng luật mới. Cần xác nhận ngày bắt đầu bán đúng ngày hiệu lực và phiên bản dùng khi chính thức thất bại chuyển thử thách.

SRS ghi chú dev cấu hình backdated cho thời gian đã qua; chưa có quy trình vận hành/phạm vi tính lại. Chưa nêu sửa/hủy cấu hình tương lai, nhiều cấu hình cùng ngày, giới hạn tháng/chỉ tiêu hoặc có cho chỉ tiêu 0 không.

PRD BR-03 chỉ nói tuần tự, SRS có cả tức thì. Guide ghi nhận hai lựa chọn SRS; không loại bỏ tức thì theo PRD. PO cần xác nhận khác biệt.

## 7. Thông báo gửi thế nào? — US-05

SRS đã có người nhận, kênh, loại và cách cấu hình lịch. HTML gộp ô lịch/kênh/người nhận cho cả bốn loại; bản Markdown làm các dòng sau trông như sai cột.

| Loại | Thời điểm | Nội dung nghiệp vụ |
| --- | --- | --- |
| Nhắc chính thức | Trước hạn theo mốc/giờ cấu hình | Báo ngày kết thúc, nhắc đạt GD tối thiểu |
| Chính thức kết thúc không đạt | Hôm sau, giờ cấu hình | Báo đã chuyển thử thách do chưa đạt |
| Nhắc thử thách | Trước hạn theo mốc/giờ cấu hình | Báo ngày kết thúc thử thách, nhắc đạt GD tối thiểu |
| Thử thách kết thúc không đạt | Hôm sau, giờ cấu hình | Báo thuộc diện xem xét chấm dứt hợp đồng |

**Người nhận:** sale các cấp thuộc đối tượng theo dõi. **Kênh:** web/app Agent.

Ví dụ kết thúc 05/10, nhắc trước 3 ngày lúc 09:00 thì gửi 02/10 lúc 09:00; thông báo kết thúc gửi 06/10 vào giờ cấu hình. Mốc nhắc dài hơn chính chu kỳ thì gửi ngày đầu kỳ.

Cần bổ sung quy tắc sale chính thức đã đạt có nhận nhắc không, thông báo thành công/thoát thử thách, nhiều mốc dồn về ngày đầu gửi mấy lần và gửi bù nếu hệ thống bỏ lỡ lịch. Các câu hỏi này không phủ nhận bốn loại thông báo đã có.

## 8. Đối chiếu với hệ thống hiện tại

### Hồ sơ sale và tổ chức

DB có `cobroker_profiles` cho hồ sơ con người, `agency_cobroker` cho quan hệ đại lý và `agent_profile_id`, `agency_profiles` cho đại lý/team/vùng, `user_registered_scope` cho đăng ký dự án.

Có nền dữ liệu sale đại lý. Danh sách O2O/Tự doanh và org chart phải xác minh thêm ở CMS/profile, không suy ra đầy đủ từ schema co-broke.

SRS dùng ID tài khoản Agent trên báo cáo. Code hiện tại mô tả chuyển đại lý có thể tạo agent-profile mới trong khi giữ con người. Cần chốt chu kỳ/GD giữ theo người hay theo tài khoản mới khi chuyển đại lý.

Profile có DRAFT/ACTIVE/INACTIVE/FROZEN/REQUEST_UPDATE; quan hệ đại lý có ACTIVE/INACTIVE. Không mapping máy móc một enum thành toàn bộ nhãn hoạt động SRS.

### Ngày bắt đầu bán

`cobroker_profiles`/`agency_cobroker` chưa có cột ngày bắt đầu bán riêng trong schema đã kiểm tra.

Có submission với tiến trình xác thực, `approve_at` và `identity_verification_history` với các lần OCR/eKYC/duyệt thủ công. Cần chọn đúng sự kiện chuyển Đã xác thực, không tự lấy ngày tạo hoặc lần eKYC bất kỳ.

SRS đã chỉ định ngày của tài khoản cũ do Chính sách cung cấp để import. Không mặc định phải suy ra toàn bộ từ lịch sử xác thực. Sau khi có ngày cũ vẫn cần chính sách/GD lịch sử để tính chu kỳ đến hiện tại.

### Giao dịch từng sale

Có `PropertySoldConsumer` nhận sale-order event từ Housing và `PropertySoldProcessor` ánh xạ SAP sang trạng thái căn. `SaleOrderChangeEvent` hiện dùng loại GD, trạng thái, mã số thuế đại lý, ngày ký cọc/HĐMB; DTO này chưa có định danh sale.

`sale_batch_units` có trạng thái bán, đại lý bán, ngày ký/bán; các cột đã kiểm tra chưa có định danh sale bán căn. Đây là dữ liệu căn theo đợt, chưa chứng minh là nguồn GD cá nhân cho 9533.

**Nghiệp vụ SRS đã xác định:** đếm GD xác nhận TTĐC/TTKQ trên SAP. **Tích hợp cần xác minh:** mã SAP tương ứng, khóa GD chống trùng, định danh sale, ngày nghiệp vụ, nguồn lịch sử. Không tự lấy `SOLD` hoặc người được phân bổ căn làm sale hưởng GD.

### Room

`RoomServiceImpl.computeRoom` có công thức theo số sale hợp lệ. `UserRegisteredScopeRepository.countValidSalesByAgency` kiểm tra quan hệ active, SALE_MEMBER, profile theo chính sách trạng thái hiện hành và đăng ký dự án active thuộc đợt. Một sale nhiều dự án trong đợt đếm một lần.

Điều kiện hiện tại chưa xét thử thách. SRS đã yêu cầu loại sale thử thách khỏi số sale tính để cấp room và khôi phục khi trở lại chính thức.

Phần cần chốt là room/căn đã cấp, room thủ công, chặn thao tác lấy căn trực tiếp và mô hình O2O/Tự doanh. Không đổi status tài khoản thành inactive để mô phỏng thử thách.

### Chu kỳ và thông báo

Chưa thấy bảng chuyên theo dõi chu kỳ/chính sách chu kỳ trong schema đã kiểm tra. Cấu hình điểm đại lý/đợt bán hiện có phục vụ nghiệp vụ khác.

Có `notification_outbox` và dịch vụ gửi/hủy theo lịch để tham khảo; cần bổ sung nghiệp vụ theo US-04/05.

Các kết luận giới hạn trong code/schema đã đọc, không khẳng định service khác thiếu dữ liệu. Không sửa DB trong quá trình review. Số lượng staging không được dùng suy ra production.

## 9. Điểm còn cần PO/BO xác nhận

### Ghi chú và mâu thuẫn trong SRS

1. **Đạt giữa kỳ:** nguyên tắc giữ đến cuối kỳ, nhưng vẫn có ghi chú BO chọn reset ngay hay không.
2. **Thử thách chưa đủ:** bảng dùng chưa đạt tối thiểu, đoạn nguyên tắc dùng không có GD; cần thống nhất khi GD > 0 nhưng < chỉ tiêu.
3. **Đạt/Chưa đạt:** bảng mô tả dữ liệu đảo điều kiện hai nhãn. Nguyên tắc/ảnh dùng Đạt khi GD >= chỉ tiêu; cần sửa lỗi nguồn.
4. **Ngày ví dụ:** có 31/11/2026 không tồn tại và 4 tháng từ 01/08 kết thúc 31/10 không khớp công thức. Không lấy ví dụ sai làm luật.
5. **Role/vùng:** tái sử dụng role 100, quản lý vùng và bộ lọc vùng còn ghi chú kiểm tra; export có cần role 25 chưa chốt.
6. **CMS:** “Ngày BC” chưa xác nhận là ngày bắt đầu bán.
7. **UI/export:** filter xếp hạng/thời gian ở tổng quan US-02, tên US-03, template xếp hạng và đặc tả lưu lịch sử export cần xác nhận.

### Tình huống chưa mô tả đủ

- GD hủy/bỏ cọc có thu hồi chỉ tiêu không? SRS đã ghi chú câu hỏi này. GD muộn/đổi người bán có sửa chu kỳ, room, thông báo đã chốt không?
- GD quản lý là cá nhân hay đội? Định danh nào nối SAP với Agent?
- Reset tức thì áp dụng cho ai và giữ GD cũ không? Tuần tự chọn luật nào khi chính thức thất bại chuyển thử thách?
- Sửa ngày bắt đầu yêu cầu tính lại chu kỳ, nhưng phạm vi hồi tố room/thông báo và policy lịch sử chưa rõ.
- Thiếu ngày, ngày tương lai hoặc thiếu cấu hình hiển thị/xử lý thế nào? Không tự kết luận không đạt.
- Chuyển đại lý/nhóm, xác thực lại hoặc tái gia nhập giữ hay bắt đầu lại kỳ?
- Ngày 29/30/31, timezone, độ trễ cập nhật GD, số ngày hiển thị sau hết kỳ và thời điểm chốt cần thống nhất.
- Căn/room đã phân bổ xử lý ra sao? Chưa có yêu cầu thu hồi tự động.
- Nhắc sale đã đạt/inactive, thông báo thành công, mốc trùng và gửi bù xử lý thế nào?

### Khác biệt với PRD

PRD chỉ mô tả áp dụng tuần tự; SRS có thêm tức thì. PRD có chữ chấm dứt vĩnh viễn; SRS xác định đề xuất, không tự sinh chu kỳ. Guide theo mô tả chi tiết SRS và giữ các khác biệt để PO xác nhận.

SRS target 31/10/2026 cho phạm vi chung; PRD cho công cụ cấu hình động tới 30/11. Cần chốt bàn giao từng đợt, không tự loại US-04 khỏi tháng 10.

## 10. Dùng tài liệu này để tiếp tục công việc

Phạm vi gồm sáu US: báo cáo, search/sort/filter, export, cấu hình, thông báo và ngày bắt đầu bán. Luồng chu kỳ và tác động room nằm trong US-01; không bỏ qua khi chỉ làm màn hình.

Dev/AI agent đọc mục 2–7 để hiểu yêu cầu, mục 8 để biết nguồn/phần còn thiếu, mục 9 để xác nhận đúng các điểm chưa chốt. Không ghi yêu cầu SRS đã có thành thiếu đặc tả; không biến ghi chú chờ BO thành luật đã duyệt.

Trước khi chốt thiết kế, cần mẫu GD SAP nối được từng sale và ví dụ BO xác nhận cho đạt giữa kỳ, thử thách chưa đủ, đổi cấu hình, GD hủy/muộn, sửa ngày và dữ liệu cũ. Chưa cần đưa schema/API chi tiết vào tài liệu phân tích này.

Tham chiếu code, tính từ gốc repo:

- Hồ sơ: `src/main/java/vn/vinhomes/cobroker/core/model/CoBrokerProfile.java`, `AgencyCobroker.java` cùng thư mục.
- Xác thực: `src/main/java/vn/vinhomes/cobroker/core/service/applicant/ApplicantEkycCompletionService.java`.
- GD: `src/main/java/vn/vinhomes/cobroker/core/consumer/PropertySoldConsumer.java`, `src/main/java/vn/vinhomes/cobroker/core/processor/PropertySoldProcessor.java`, `src/main/java/vn/vinhomes/cobroker/core/dto/distribution/event/SaleOrderChangeEvent.java`.
- Sale hợp lệ/room: `src/main/java/vn/vinhomes/cobroker/core/repository/UserRegisteredScopeRepository.java`, `src/main/java/vn/vinhomes/cobroker/core/service/distribution/impl/RoomServiceImpl.java`.
- Thông báo: `src/main/java/vn/vinhomes/cobroker/core/service/notification/outbox/NotificationOutboxService.java`.

[Phân tích PRD riêng](BDSKD-9533-phan-tich-PRD.md) giữ góc nhìn PRD. Guide này đã hiệu chỉnh theo SRS trực tiếp. Chưa đọc demo SharePoint, Excel template hoặc Q&A bên ngoài trang, nên không coi chúng là nguồn đã kiểm chứng.
