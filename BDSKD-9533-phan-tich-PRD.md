# BDSKD-9533 — Hiểu yêu cầu theo dõi kết quả bán hàng

Ngày cập nhật: 05/10/2026.

Nguồn chính là PRD “Theo dõi kết quả bán hàng - ngừng hợp tác”, bản 0.1 ngày 23/09/2026. Phần đối chiếu SRS dùng [trang Confluence bản 7](https://vin3s.atlassian.net/wiki/spaces/BMAS/pages/3174532173), cập nhật 01/10/2026. Cả hai tài liệu đang chờ phê duyệt.

**Đọc file này để hiểu mục tiêu và luồng nghiệp vụ.** Chi tiết sáu US, các cột báo cáo, quyền và hiện trạng hệ thống nằm trong [guide theo SRS](BDSKD-9533-dev-guide.md). Những phần SRS bổ sung được ghi rõ, không coi là nội dung đã có trong PRD.

## 1. PO muốn giải quyết vấn đề gì?

Hiện quản lý phải theo dõi thủ công sale nào bán được, sale nào lâu không có giao dịch và ai cần xem xét ngừng hợp tác. Vì vậy việc đôn đốc chậm, khó thống nhất kết quả và hạn mức căn vẫn có thể được dành cho lực lượng bán hàng không hiệu quả.

PO muốn hệ thống trả lời ba câu hỏi cho **từng sale**:

1. Sale đang được theo dõi trong khoảng thời gian nào?
2. Trong khoảng đó, sale đã đạt số giao dịch yêu cầu chưa?
3. Nếu chưa đạt, sale còn cơ hội bán hay đã thuộc diện đề xuất ngừng hợp tác?

Đây là đánh giá theo **số giao dịch trong từng chu kỳ**, không phải xếp hạng doanh thu hoặc so sánh sale với nhau.

## 2. Sale sẽ trải qua những giai đoạn nào?

### Giai đoạn bán chính thức

Sale được một khoảng thời gian để đạt chỉ tiêu. Ví dụ đại lý được 4 tháng để đạt 1 giao dịch.

Nếu cuối kỳ đạt, sale tiếp tục một kỳ bán chính thức mới. Nếu không đạt, sale chuyển sang giai đoạn cảnh báo.

### Giai đoạn cảnh báo / thử thách

Sale có thêm thời gian để bán, nhưng **không được tính để cấp hạn mức căn**. PRD gọi giai đoạn này là cảnh báo; SRS gọi là thử thách. Đây là cùng một giai đoạn, không phải hai loại khác nhau.

Nếu đạt trong giai đoạn này, sale trở lại bán chính thức từ ngày hôm sau. Nếu hết thời gian vẫn không đạt, sale thuộc diện đề xuất chấm dứt hợp tác.

**“Đề xuất chấm dứt” là kết quả đánh giá.** SRS không yêu cầu tự khóa tài khoản hoặc tự chấm dứt hợp đồng. PRD có chỗ viết “chấm dứt vĩnh viễn”, nên PO cần thống nhất cách diễn đạt và hành động tiếp theo.

## 3. Ví dụ từ lúc bắt đầu đến lúc có kết quả

Giả sử sale A bắt đầu bán ngày 01/01/2026. Chính sách minh họa là 4 tháng chính thức, 6 tháng thử thách, mỗi giai đoạn cần 1 giao dịch.

**Sale A có giao dịch trong kỳ chính thức:**

- Từ 01/01 đến 30/04, A đang trong kỳ chính thức.
- Ngày 15/03, A đạt 1 giao dịch và được ghi nhận đạt yêu cầu.
- Theo phần nguyên tắc SRS, A tiếp tục kỳ này đến 30/04; ngày 01/05 mới mở kỳ chính thức tiếp theo.

**Sale A không đạt trong kỳ chính thức:**

- Hết 30/04, A chưa đạt nên chuyển sang thử thách từ 01/05 đến 31/10.
- Trong thử thách, A không được tính để cấp hạn mức căn.
- Nếu ngày 10/06 A đạt chỉ tiêu, thử thách đóng trong ngày 10/06; ngày 11/06 mở kỳ chính thức mới.
- Nếu hết 31/10 A vẫn không đạt, A thuộc diện đề xuất chấm dứt; SRS không tự mở thêm kỳ tiếp theo.

Các con số trên chỉ để giải thích. Chính sách thực tế cần có chỉ tiêu và ngày hiệu lực được duyệt.

Hai chỗ nguồn vẫn cần xác nhận: giữ đến cuối kỳ chính thức hay reset ngay khi đạt; hết thử thách có GD nhưng chưa đủ chỉ tiêu xử lý thế nào. SRS có hướng mô tả rõ hơn PRD, nhưng còn ghi chú/mâu thuẫn; xem mục 7.

## 4. Người dùng cần làm được gì?

**Khối Chính sách / Kinh doanh** xem toàn bộ báo cáo và thiết lập chính sách riêng cho Đại lý, O2O, Tự doanh. Chính sách gồm thời gian chính thức, thời gian thử thách, số GD và ngày bắt đầu áp dụng.

**Admin đại lý / Quản lý kinh doanh** xem sale thuộc bộ phận mình quản lý. Họ cần biết còn bao nhiêu ngày, đã có bao nhiêu GD và đang ở giai đoạn nào để đôn đốc hoặc lập danh sách xử lý. Có bộ lọc và xuất Excel.

**Sale** nhận nhắc nhở trước khi hết kỳ và thông báo kết quả trên web/app Agent.

Ngoài các màn hình, hệ thống phải tự cập nhật GD, xác định kết quả và chuyển kỳ. Nếu chỉ làm bảng hiển thị mà chưa có dữ liệu/luật tính đúng, chưa đáp ứng mục tiêu PRD.

PRD loại khỏi đợt này việc reset thủ công riêng từng sale và màn hình xem toàn bộ lịch sử chu kỳ. SRS vẫn cho sửa ngày bắt đầu bán và tự tính lại chu kỳ; cần hiểu đây là hai thao tác khác nhau, dù có thể ảnh hưởng cùng kết quả.

## 5. Hệ thống hiện tại hỗ trợ được đến đâu?

Phần này dựa trên code staging `31444ce6` và DB `cobroker_db` đã kiểm tra chỉ đọc ngày 05/10/2026. Chưa kiểm tra DB các service khác.

**Có dữ liệu hồ sơ và quan hệ sale đại lý.** Hệ thống biết hồ sơ con người, tài khoản Agent, đại lý và dự án đăng ký. Nguồn O2O/Tự doanh cần xác minh thêm ở CMS/profile. Khi chuyển đại lý, tài khoản Agent có thể đổi dù vẫn là cùng người; PRD/SRS chưa nói rõ chu kỳ phải giữ hay bắt đầu lại.

**Có lịch sử xác thực, nhưng chưa có cột ngày bắt đầu bán riêng ở các bảng profile/link đã kiểm tra.** Không tự dùng ngày tạo hồ sơ làm ngày bắt đầu bán. SRS đã hướng dẫn ngày cho sale cũ do Chính sách cung cấp để import.

**Có dữ liệu căn bán theo đại lý, chưa chứng minh đủ GD từng sale.** Luồng sale order hiện được code sử dụng có đại lý và trạng thái/ngày bán căn, chưa có định danh sale trong DTO đã kiểm tra. Biết đại lý bán được một căn chưa đủ để biết sale nào được tính chỉ tiêu.

**Có room tính theo số sale hợp lệ.** Điều kiện hiện tại chưa xét chu kỳ thử thách. Thêm yêu cầu này có thể làm room đại lý giảm; cần làm rõ căn đã phân bổ xử lý thế nào.

Vì vậy, hai việc cần xác minh dữ liệu trước là **ngày bắt đầu bán** và **GD được ghi nhận cho đúng từng sale**. Guide có tên bảng và file code để dev kiểm tra tiếp.

## 6. SRS đã bổ sung gì cho PRD?

Không nên tiếp tục hỏi lại toàn bộ những phần dưới đây như thể chưa có yêu cầu:

| Nội dung | SRS đã mô tả |
| --- | --- |
| Đối tượng / quyền | Nhóm sale, phần lớn role và phạm vi xem theo tổ chức |
| Ngày bắt đầu bán | Đại lý tự ghi khi đã xác thực; role 11/100 được sửa; role 64 chỉ xem; O2O/Tự doanh nhập CMS; tài khoản cũ import |
| Mốc chu kỳ | Ngày kết thúc = ngày bắt đầu + số tháng - 1 ngày; kỳ tiếp bắt đầu hôm sau |
| Thoát thử thách | Đạt đủ GD yêu cầu, đóng trong ngày đạt và mở chính thức hôm sau |
| Báo cáo | Các cột và cách hiển thị chính thức/thử thách gần nhất |
| Excel | Xuất toàn bộ kết quả lọc, tối đa 50.000 dòng |
| Cấu hình | Có tức thì và tuần tự; UI chỉ chọn ngày hiệu lực tương lai |
| Thông báo | Bốn loại, gửi web/app theo mốc ngày và giờ cấu hình |

SRS không phải đã giải quyết mọi câu hỏi. Một số yêu cầu bổ sung còn có ghi chú chờ BO hoặc khác với PRD.

## 7. Những điểm thật sự còn cần chốt

### Giao dịch nào được tính?

SRS xác định GD đã xác nhận TTĐC/TTKQ trên SAP. Nhóm tích hợp cần xác minh mã trạng thái, ngày nghiệp vụ và field định danh sale tương ứng. Cần làm rõ một GD được đếm một lần thế nào, GD của quản lý là cá nhân hay đội, GD hủy/đến muộn có thay đổi kết quả cũ không.

Ví dụ: xác nhận ngày 30/04 nhưng nhận dữ liệu ngày 02/05 thì có tính lại kỳ kết thúc 30/04 và hoàn tác thử thách/room không?

### Đạt thì chuyển kỳ lúc nào?

Phần nguyên tắc SRS nói đạt giữa kỳ chính thức vẫn giữ đến cuối kỳ, nhưng còn ghi chú BO xác nhận reset ngay hay không.

Trong thử thách, điều kiện đạt đã là đủ chỉ tiêu. Tuy nhiên đoạn hết hạn viết “không có GD”, còn bảng phân loại viết “chưa đạt”. Với chỉ tiêu 2 và mới có 1 GD, cần thống nhất vẫn bị coi là không đạt.

### Chính sách mới tác động đến sale đang chạy thế nào?

PRD chỉ nói hoàn tất kỳ cũ rồi dùng luật mới. SRS thêm lựa chọn reset tức thì. PO cần xác nhận lựa chọn này, sale nào bị reset và GD cũ có giữ không.

Với tuần tự, cần trả lời kỳ chính thức cũ thất bại thì thử thách tiếp theo dùng luật cũ hay mới.

### Room và kết quả cuối có tác động gì?

Loại sale thử thách khỏi số sale được tính để cấp room đã là yêu cầu. Phần còn thiếu là căn/room đã cấp có thu hồi không, có chặn sale trực tiếp lấy căn không và áp dụng thế nào với room thủ công/O2O/Tự doanh.

Đề xuất chấm dứt cần có cách xử lý tiếp rõ ràng. Không tự thêm hành động khóa tài khoản từ tên nhãn.

### Sale cũ và sửa ngày được tính lại ra sao?

SRS đã nêu nguồn ngày sale cũ và yêu cầu tính lại khi sửa ngày. Cần có GD/chính sách lịch sử tương ứng và quyết định về tác động tới room/thông báo. Sale thiếu dữ liệu không được tự kết luận là không đạt.

Các câu hỏi chi tiết về role vùng, template Excel, ngày cuối tháng và thông báo còn lại nằm trong mục 9 của guide, tránh lặp một danh sách dài ở đây.

## 8. Phạm vi bàn giao cần thống nhất

PRD dự kiến UAT 25–30/10/2026, go-live 31/10/2026; riêng công cụ cấu hình động có thể đến 30/11. SRS đặt target 31/10 cho phạm vi chung.

Cần chốt đợt tháng 10 bàn giao những gì. Nếu cấu hình UI làm sau, vẫn phải có chính sách ban đầu được duyệt để tính chu kỳ, gửi thông báo và xác định room.

Khi nghiệm thu, cần dùng ví dụ sale cụ thể để kiểm tra ngày bắt đầu, GD được tính, giai đoạn và tác động room. Kiểm tra bảng có hiển thị đúng giao diện chưa đủ để chứng minh kết quả đánh giá đúng.
