# BDSKD-9531 — Phân tích PRD xếp hạng sale theo doanh số

Cập nhật 05/10/2026. Nguồn: [PRD bản 0.1 ngày 23/09/2026](PRD_Xep_hang_Sale.odt), chờ phê duyệt. Đối chiếu [SRS local](<SRS - Xếp hạng Sale theo doanh số.md>); chưa xác minh phiên bản online. [Mục lục và giới hạn nguồn](README.md).

## 1. PO muốn giải quyết việc gì?

Hiện việc tổng hợp thành tích và công nhận Kim cương/Bạch kim/Vàng làm thủ công. PO muốn có bảng số liệu theo từng sale/quý, tiêu chí thống nhất, badge trên Agent và chứng nhận tải được.

Hệ thống cần trả lời: sale có bao nhiêu GD và doanh số trong quý; điểm sau khi tính thêm quý trước là bao nhiêu; sale đạt hạng nào sau khi xét ngưỡng và quota. Có điểm cao chưa tự đủ hạng nếu không đạt điều kiện số lượng được công nhận.

## 2. Phạm vi PRD

| Mã | Yêu cầu |
| --- | --- |
| R-01 | Báo cáo số lượng và giá trị GD theo sale/quý, bộ lọc |
| R-02 | Điểm có trọng số hai quý; ví dụ 30% quý trước + 70% quý này |
| R-03 | Kim cương/Bạch kim/Vàng theo ngưỡng và quota; lưu thứ hạng số cho sale còn lại |
| R-04 | Badge công nhận và chứng nhận điện tử sau chốt quý |
| R-05 | Bắt buộc sale/đại lý trên GD; khóa attribution sau ký HĐMB |
| R-06 | Phân quyền theo toàn hệ thống/đại lý/bộ phận/vùng |
| R-07 | Giữ sale inactive, có bộ lọc hoạt động |
| R-08 | Xuất Excel đầy đủ cột/dòng trong phạm vi lọc |

Không xếp hạng tháng/năm, không tạo bảng hạng riêng theo dự án/đại lý. Lọc dự án để xem số liệu không đồng nghĩa chấm lại hạng riêng của dự án.

R-05 thuộc nguồn quản lý GD: service xếp hạng nhận attribution đã được xác nhận và không cung cấp API sửa sale trên GD. Để hoàn tất yêu cầu end-to-end, bên pipeline/nguồn GD cần bảo đảm gắn và khóa attribution; consumer riêng không thể chứng minh trường đã bị khóa ở hệ thống nguồn.

## 3. Công thức và ví dụ

```text
Điểm doanh số quý Q = w_prev × doanh số Q−1 + w_current × doanh số Q
```

Ví dụ với tỷ lệ 30/70: A có doanh số Q2 = 10 tỷ, Q3 = 20 tỷ thì điểm Q3 = 17 tỷ. Số GD Q3 vẫn là số GD thực trong Q3, không cộng trọng số vào cột Số GD.

PRD còn yêu cầu điểm theo số lượng GD; SRS chi tiết tập trung vào điểm/hạng doanh số. Không tự tạo engine hạng số lượng thứ hai trước khi PO xác nhận phạm vi cuối cùng.

Sale mới không có kết quả quý trước thì phần quý trước bằng 0. Khác với dữ liệu lịch sử chưa nhập đủ: trường hợp thiếu nguồn phải được đánh dấu chờ dữ liệu, không kết luận bằng 0.

## 4. Luồng từ GD đến chứng nhận

1. Nguồn GD gắn sale/đại lý, kiểm tra hợp lệ và khóa theo yêu cầu khi ký HĐMB.
2. Pipeline phát dữ liệu đủ điều kiện; service mới nhận và lưu ledger GD.
3. Trong quý, service tổng hợp số GD/doanh số và điểm tạm tính; chưa công nhận hạng chính thức.
4. Sau cuối quý và khi nguồn đã đủ dữ liệu, service tính toàn tập, xét ngưỡng/quota, publish kết quả quý.
5. Web/app lấy badge của quý trước; sale đạt hạng tải chứng nhận.
6. Quản lý xem/lọc/export theo quyền; thao tác đọc không chấm lại hạng trong tập vừa lọc.

## 5. Những điểm SRS khác hoặc cụ thể hơn PRD

| Chủ đề | PRD | SRS chi tiết / hướng tài liệu |
| --- | --- | --- |
| Cơ sở chấm điểm | Cả số lượng và doanh số | Cột/rule chi tiết chủ yếu doanh số; hạng số lượng cần PO xác nhận |
| Thời điểm GD | Ngày ký VBCN/HĐMB SAP | KH xác nhận HĐMB/VBCN; phải chốt mốc chính xác với pipeline |
| GD bị hủy sau ghi nhận | Chưa diễn giải rõ | Vẫn tính; không lấy cancellation thông thường để trừ GD |
| Thời gian báo cáo | Theo quý | Bắt đầu Q3/2026, chọn tối đa 4 quý, mặc định quý trước |
| Đồng điểm tại quota | Không rõ | Ghi chú cho cả hai sale nhận hạng; cần chốt cách áp dụng và quota tầng tiếp |
| Export | Excel | Tối đa 50.000 dòng, vượt thì tự cắt; role chỉ 21 không được xuất |
| Badge/chứng nhận | Sau chốt quý | Quý này dùng quý trước, reset đầu quý; file PNG/JPEG |
| Cấu hình | Tỷ lệ và ngưỡng/quota động | Tên ba hạng cố định; UI chỉ cấu hình quý hiện tại/tương lai |
| Timeline | Go-live 31/10; cấu hình động muộn nhất 30/11 | Target chung 31/10; phải chốt kế hoạch US-04 |

PRD yêu cầu chuyển đại lý không reset tích lũy; GD giữ đại lý tại thời điểm phát sinh. Đây là yêu cầu phải thiết kế mapping người ổn định nếu Agent ID đổi; không dùng UNIQUE Agent ID của 9533 để mặc định tách điểm.

## 6. Kiến trúc dữ liệu

Hồ sơ/role/tổ chức lấy từ profile-mw, GD từ Kafka sale-pipeline. Service có DB riêng cho policy theo quý, ledger GD, tổng hợp, điểm/hạng, audit/task và chứng nhận. Không dùng DB hoặc code core-broker, không sửa user.sale_score hay sales_member_tier của profile-mw để ghi hạng.

Các chỉ số tạm tính và kết quả chính thức có trạng thái riêng. Badge/chứng nhận chỉ đọc kết quả quý đã publish. Thiếu dữ liệu hoặc chưa chốt quý không tương đương Không xếp hạng.

## 7. Những điểm cần PO/nguồn GD xác nhận

- Có triển khai hạng theo số lượng GD trong MVP không? Dữ liệu dự án và bộ lọc có phải bắt buộc?
- Mốc quý là KH xác nhận hay ngày ký SAP? Một GD có hai mốc HĐMB/VBCN được đếm thế nào?
- Xét quota toàn ba kênh chung hay từng audience theo cấu hình? Không chia theo bộ phận/dự án người dùng đang xem.
- Ngưỡng dùng điểm weighted hay doanh số riêng quý? Cách xử lý đồng điểm, quota bỏ trống/không đủ người và quota hạng tiếp theo?
- Điểm/attribution qua đổi Agent ID và đổi audience; phạm vi quyền xem GD đại lý cũ?
- Dữ liệu T−1 đến sau cuối quý được chốt khi nào? Khi sửa dữ liệu đã chốt có thu hồi badge/cert không?
- Khi quý mới chưa có kết quả quý trước, UI reset badge hiển thị gì; certificate cũ còn tải được không?

[Tài liệu SRS](BDSKD-9531-xep-hang-sale-theo-doanh-so-phan-tich-SRS.md) giải thích từng US. [TDD](BDSKD-9531-xep-hang-sale-theo-doanh-so-TDD.md) ghi rõ phần có thể implement trước và phần chưa được bật khi chưa chốt luật.
