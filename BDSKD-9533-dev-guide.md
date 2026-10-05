# BDSKD-9533 — Hướng dẫn yêu cầu cho dev và AI agent

Ngày review: 05/10/2026. Code đối chiếu: nhánh `staging`, commit `31444ce6`.

## 1. Mục đích và trạng thái tài liệu

Theo dõi hiệu quả bán hàng của từng sale bằng chu kỳ bán chính thức và chu kỳ thử thách. Hệ thống thống kê giao dịch, chuyển chu kỳ, gửi nhắc nhở và cung cấp dữ liệu để xác định sale được tính vào hạn mức căn (room).

Tài liệu này tổng hợp và review hai tài liệu trong cùng thư mục:

- `PRD-Theo dõi kết quả bán hàng - ngừng hợp tác.odt`, phiên bản 0.1, ngày 23/09/2026, chờ phê duyệt.
- `SRS - Theo dõi chu kỳ bán hàng & cảnh báo ngưng hợp tác.md`, trạng thái Under approve, gồm US-01 đến US-06.

Chưa kiểm tra các thiết kế, file Excel và Q&A được dẫn link bên ngoài. Tài liệu này không thay thế quyết định phê duyệt của PO/BO.

Quy ước đọc:

| Nhãn | Ý nghĩa | Dev/AI agent được làm gì? |
| --- | --- | --- |
| Yêu cầu nguồn | Nội dung đã được mô tả trong PRD/SRS | Triển khai sau khi xử lý các điểm chặn liên quan |
| Đề xuất kỹ thuật | Cách triển khai được đề xuất trong bản review | Có thể dùng làm thiết kế, không coi là chính sách PO đã duyệt |
| Cần chốt Qxx | Thiếu thông tin hoặc có mâu thuẫn | Không tự chọn một luật rồi đưa vào production |

**Đánh giá:** đủ để phân rã công việc và thiết kế sơ bộ; chưa đủ để chốt logic production. Các điểm quan trọng nhất là nguồn giao dịch SAP, chuyển chu kỳ, áp dụng chính sách mới, dữ liệu cũ và tác động đến room.

## 2. Phạm vi

### 2.1. Những phần cần làm

1. Bảng theo dõi chu kỳ bán hàng trên web, có phân quyền theo tổ chức.
2. Tìm kiếm, sắp xếp, lọc và phân trang.
3. Xuất Excel theo toàn bộ kết quả lọc trong phạm vi được phép.
4. Cấu hình thời gian, chỉ tiêu, ngày hiệu lực và lịch thông báo theo nhóm đối tượng.
5. Tự động tính giao dịch và chuyển trạng thái chu kỳ.
6. Thông báo trên web/app Agent.
7. Ngày bắt đầu bán: tự ghi nhận cho sale đại lý; nhập/cập nhật từ CMS cho O2O/Tự doanh; nhập dữ liệu cũ.
8. Tích hợp điều kiện chu kỳ vào việc tính room, có phối hợp hệ thống liên quan.

### 2.2. Ngoài phạm vi theo nguồn

- Reset chu kỳ thủ công cho từng sale.
- Màn hình xem toàn bộ lịch sử chu kỳ; PRD nói IT xuất dữ liệu khi có yêu cầu.

Vẫn cần lưu lịch sử nội bộ để kiểm chứng kết quả, áp dụng chính sách và xử lý đồng bộ lại. Đây là đề xuất kỹ thuật, không thêm màn hình lịch sử cho người dùng.

### 2.3. Mốc bàn giao cần xác nhận

PRD đặt UAT 25–30/10/2026, go-live 31/10/2026; riêng công cụ cấu hình động có thể muộn nhất 30/11/2026. SRS đặt target 31/10/2026 và đưa cấu hình động vào phạm vi chung. Cần chốt phần nào bàn giao tháng 10 và cấu hình tạm thời nếu UI cấu hình làm sau (Q12).

## 3. Đối tượng và quyền

### 3.1. Sale được theo dõi

| Nhóm | Điều kiện trong SRS | Nguồn dữ liệu |
| --- | --- | --- |
| Đại lý | Thuộc đại lý và có role Sale member (21) | Agent/co-broke |
| O2O | Thuộc khối O2O và có role 21, 20, 23 hoặc 60 | CMS/org chart và profile |
| Tự doanh | Thuộc khối Tự doanh và có role 21, 20, 23 hoặc 60 | CMS/org chart và profile |

ID khối `1060`/`1132` trong SRS là link môi trường staging. Không hardcode cho mọi môi trường nếu chưa xác nhận mapping.

Chưa có luật khi một người có nhiều role/nhóm, chuyển đại lý, chuyển O2O/Tự doanh hoặc tái gia nhập (Q09). Phải thống nhất định danh sale giữa SAP, IAM, CMS và co-broke; không dùng số điện thoại làm khóa duy nhất nếu chưa có hợp đồng dữ liệu.

### 3.2. Ma trận quyền từ SRS

| Vai trò | Xem dữ liệu | Cấu hình | Sửa ngày bắt đầu bán |
| --- | --- | --- | --- |
| Admin hệ thống (11) | Toàn bộ | Có | Có, cho sale đại lý |
| Quản lý đại lý (100) | Toàn bộ, còn ghi chú cần check quyền | Có | Có, cho sale đại lý |
| Admin đại lý (64) | Sale thuộc đại lý quản lý | Không được nêu | Chỉ xem |
| Quản lý vùng chủ quản đại lý | Sale thuộc các đại lý trong vùng | Không được nêu | Không được nêu |
| Giám đốc miền (60) | Sale thuộc miền quản lý | Không được nêu | Không được nêu |
| Giám đốc phòng (23) | Sale thuộc phòng quản lý | Không được nêu | Không được nêu |
| Trưởng nhóm (20) | Sale thuộc nhóm quản lý | Không được nêu | Không được nêu |

Quyền export có cần role `USER_EXPORTER (25)` hay không còn mở. Role quản lý vùng và nguồn xác định vùng chưa chốt (Q08).

Đề xuất kỹ thuật: backend kiểm tra quyền và phạm vi trên mọi API list/detail/export/update/configuration. Bộ lọc người dùng chỉ được thu hẹp phạm vi được phép; không được mở rộng phạm vi. Cần chốt cách hợp nhất quyền khi user có nhiều role.

## 4. Khái niệm dữ liệu

| Khái niệm | Ý nghĩa |
| --- | --- |
| Ngày bắt đầu bán | Ngày gốc dùng để mở chu kỳ bán đầu tiên |
| Chu kỳ bán chính thức | Sale đang trong giai đoạn bán có khả năng được tính vào room theo các điều kiện khác của hệ thống |
| Chu kỳ thử thách | Sale tiếp tục bán nhưng không được tính để cấp hạn mức căn |
| Chỉ tiêu | Số giao dịch hợp lệ cần đạt trong một chu kỳ |
| Giao dịch hợp lệ | Cần chốt trạng thái/mốc SAP; hiện nguồn mô tả chưa thống nhất |
| Đề xuất chấm dứt hợp tác | Nhãn kết quả sau thử thách thất bại; SRS không yêu cầu tự khóa tài khoản hay tự chấm dứt hợp đồng |
| Chính sách | Bộ tham số có phiên bản, nhóm áp dụng và ngày hiệu lực |

Trạng thái hoạt động tài khoản và phân loại chu kỳ là hai thuộc tính riêng. Không dùng `INACTIVE` để đại diện cho “Chu kỳ thử thách”. SRS quy định chu kỳ vẫn chạy khi trạng thái hoạt động của sale thay đổi.

## 5. Luồng nghiệp vụ chu kỳ

### 5.1. Các trạng thái đề xuất cho thiết kế

Tên kỹ thuật bên dưới là đề xuất; nhãn hiển thị lấy theo SRS.

| Mã đề xuất | Nhãn hiển thị | Điều kiện |
| --- | --- | --- |
| OFFICIAL_NOT_MET | Chu kỳ bán - Chưa đạt yêu cầu | Đang bán chính thức và số GD < chỉ tiêu |
| OFFICIAL_MET | Chu kỳ bán - Đạt yêu cầu | Đang bán chính thức và số GD >= chỉ tiêu |
| CHALLENGE | Chu kỳ thử thách | Chu kỳ chính thức trước đó kết thúc không đạt |
| TERMINATION_PROPOSED | Đề xuất chấm dứt hợp tác | Chu kỳ thử thách kết thúc không đạt; cần Q02 xác nhận điều kiện tổng quát |

Thiếu ngày bắt đầu bán/chính sách hoặc ngày bắt đầu bán còn ở tương lai: chưa được nguồn quy định nhãn và hành vi. Không tự đưa sale này vào thử thách (Q10).

### 5.2. Quy tắc chuyển trạng thái

| Đang ở đâu? | Sự kiện/điều kiện | Hành vi trong nguồn | Điểm cần chốt |
| --- | --- | --- | --- |
| Chưa có chu kỳ | Có ngày bắt đầu bán và chính sách áp dụng | Mở chu kỳ chính thức đầu tiên từ ngày bắt đầu bán | Q10, Q11 cho dữ liệu cũ |
| Chính thức, chưa đạt | Số GD đạt chỉ tiêu | Gán “Đạt yêu cầu”, bán tiếp đến hết chu kỳ | Q01: nguồn còn ghi chú BO confirm |
| Chính thức, đã đạt | Hết ngày kết thúc | Ngày hôm sau mở chu kỳ chính thức mới, đếm GD lại từ đầu | Q01 |
| Chính thức, chưa đạt | Hết ngày kết thúc | Ngày hôm sau chuyển thử thách và loại khỏi điều kiện tính room | Q06 |
| Thử thách | Số GD đạt chỉ tiêu | Đóng thử thách trong ngày đạt; hôm sau mở chính thức mới và khôi phục điều kiện room | Q02, Q04, Q06 |
| Thử thách | Hết hạn và chưa đạt | Dừng sinh chu kỳ; đề xuất chấm dứt | Q02: đoạn nguồn ghi “vẫn không có GD” |
| Đề xuất chấm dứt | Có GD mới hoặc chính sách mới | SRS nói không tự reset; chưa có luật phục hồi ngoại lệ | Q03 |

Một giao dịch đã dùng cho chu kỳ trước không được tự cộng lại cho chu kỳ mới. Cách xử lý GD đến muộn, GD hủy hoặc ngày bắt đầu bán bị sửa phải được chốt trước khi viết logic tái tính (Q04, Q05, Q07).

### 5.3. Thời gian

Yêu cầu nguồn:

```text
endDate = startDate + số tháng cấu hình - 1 ngày
nextStartDate = previousEndDate + 1 ngày
remainingDays = endDate - ngày hiện tại
```

Ví dụ chuẩn: chu kỳ 3 tháng bắt đầu 01/01/2026, kết thúc 31/03/2026; chu kỳ kế tiếp bắt đầu 01/04/2026.

Đề xuất kỹ thuật cần xác nhận:

- Dùng ngày nghiệp vụ theo `Asia/Ho_Chi_Minh`; timestamp tích hợp có timezone rõ ràng.
- Ngày kết thúc vẫn được tính GD tới hết ngày. `remainingDays = 0` trong ngày kết thúc không có nghĩa chuyển trạng thái ngay đầu ngày.
- Số ngày còn lại của chu kỳ đã kết thúc hiển thị 0 thay vì số âm.
- Tính số ngày từ ngày kết thúc, không lưu bộ đếm rồi trừ 1 mỗi lần job chạy.
- Với ngày 29/30/31, phải chốt cách cộng tháng và chu kỳ kế tiếp. Ví dụ `31/01/2026 + 1 tháng - 1 ngày` có thể cho 27/02/2026 nếu dùng phép cộng tháng kiểu clamp. Không tự thay bằng cuối tháng nếu PO chưa duyệt (Q13).

### 5.4. Ví dụ để PO/QA xác nhận

Giả sử chính sách 3 tháng chính thức, 3 tháng thử thách, mỗi chu kỳ cần 1 GD:

1. Bắt đầu bán 01/01/2026 → chính thức 01/01–31/03.
2. Có GD hợp lệ ngày 10/02 → đạt yêu cầu nhưng giữ chu kỳ hiện tại; 01/04 mở chính thức mới.
3. Nếu không đạt đến hết 31/03 → thử thách 01/04–30/06.
4. Trong thử thách đạt chỉ tiêu ngày 10/04 → đóng thử thách 10/04; mở chính thức mới 11/04.
5. Nếu hết 30/06 vẫn chưa đạt → từ 01/07 đề xuất chấm dứt và không sinh chu kỳ tiếp.

Ví dụ 2 phụ thuộc Q01; ví dụ 4–5 phụ thuộc Q02 và luật nhận GD muộn.

## 6. Giao dịch SAP — hợp đồng dữ liệu phải chốt

SRS mô tả đếm GD tới bước KH xác nhận TTĐC/TTKQ trên SAP. PRD mục 8.1 lại mô tả thời điểm ký VBCN/HĐMB. Đây là khác biệt có thể làm sai toàn bộ phân loại.

Trước khi tích hợp, phải có câu trả lời và mẫu payload cho:

1. Trạng thái SAP nào tạo một GD hợp lệ? TTĐC hay TTKQ là hai mốc của cùng GD hay hai GD?
2. Đếm theo hợp đồng, booking, căn hay mã GD? Khóa chống trùng là gì?
3. Field định danh sale, field ngày nghiệp vụ và field thời gian cập nhật là gì?
4. Sale cấp quản lý tính GD cá nhân hay tổng GD của đội? Một GD có được tính cho nhiều sale không?
5. Hủy/bỏ cọc/chuyển sale làm thay đổi chỉ tiêu thế nào?
6. GD nghiệp vụ ngày 31/03 nhưng đồng bộ ngày 02/04 thuộc chu kỳ nào? Có hoàn tác chuyển thử thách, room và thông báo không?
7. Sau khi đã đề xuất chấm dứt mới nhận GD thuộc thử thách cũ thì xử lý thế nào?
8. Nguồn lấy lịch sử, cơ chế cập nhật, độ trễ và khả năng đối soát là gì?

Đề xuất kỹ thuật: lưu các GD đã chuẩn hóa với ID nguồn, sale, ngày nghiệp vụ, trạng thái và phiên bản/thời điểm cập nhật; xử lý replay không tăng số GD. Không chỉ lưu một số tổng mà không có cách truy nguyên.

## 7. Cấu hình chính sách — US-04

### 7.1. Nội dung cấu hình

| Trường | Yêu cầu nguồn / phần cần chốt |
| --- | --- |
| Ngày hiệu lực | UI chỉ chọn ngày tương lai, không chọn hôm nay/quá khứ |
| Cơ chế áp dụng | SRS có tức thì và tuần tự; PRD BR-03 chỉ cho tuần tự → Q03 |
| Đối tượng | Đại lý, O2O, Tự doanh; mỗi đối tượng chỉ thuộc một box trong cùng cấu hình |
| Thời gian chính thức | Số tháng, bắt buộc |
| Chỉ tiêu chính thức | Số GD cần đạt, bắt buộc |
| Thời gian thử thách | Số tháng, bắt buộc |
| Chỉ tiêu thử thách | Số GD cần đạt, cần xác nhận có khác chỉ tiêu chính thức không |
| Nhắc trước | Nhiều mốc số ngày và giờ:phút; tùy chọn |
| Báo kết thúc | Giờ:phút gửi ngày tiếp theo sau khi chu kỳ kết thúc; tùy chọn |
| Lịch sử cấu hình | Xem các cấu hình đã lưu |

Các ví dụ 4–6 tháng cho đại lý và 3–3 tháng cho O2O/Tự doanh chưa đủ làm cấu hình production nếu chưa có chính sách được duyệt và ngày hiệu lực.

### 7.2. Áp dụng tuần tự

Theo SRS: sale bắt đầu bán sau ngày hiệu lực dùng luật mới; sale đã bán hoàn thành chu kỳ hiện tại rồi dùng luật mới.

Cần xác nhận “chu kỳ hiện tại” là một giai đoạn chính thức/thử thách hay toàn bộ cặp chính thức + thử thách. Ví dụ chính thức cũ thất bại sau ngày luật mới có hiệu lực: thử thách dùng luật cũ hay mới? Sale bắt đầu đúng ngày hiệu lực cũng cần định nghĩa rõ.

### 7.3. Áp dụng tức thì

SRS nói reset tất cả sale về đầu chu kỳ khi luật mới bắt đầu; PRD không cho reset ngang chu kỳ. Chưa rõ có bao gồm sale thử thách và sale đã đề xuất chấm dứt không, GD đã ghi nhận giữ hay bỏ, room khôi phục khi nào. Không triển khai luật này theo suy đoán (Q03).

### 7.4. Đề xuất thiết kế

- Lưu phiên bản chính sách gắn với từng chu kỳ; không sửa số tháng/chỉ tiêu của chu kỳ cũ bằng cách đọc cấu hình mới nhất.
- Backend kiểm tra đối tượng trùng, kiểu số, khoảng giá trị, giờ hợp lệ và ngày hiệu lực. PO cần chốt min/max và có cho chỉ tiêu 0 không.
- Cần quy định sửa/hủy cấu hình tương lai, nhiều cấu hình cùng ngày, thứ tự hiệu lực, nhóm không có cấu hình và cấu hình backdated cho dữ liệu cũ.
- Backdated phải có quy trình vận hành và dấu vết thay đổi; ghi chú “dev cấu hình backdated” chưa phải đặc tả đủ để chạy.

## 8. Ngày bắt đầu bán — US-06

| Nhóm | Yêu cầu nguồn |
| --- | --- |
| Đại lý mới | Tự ghi ngày hoàn thành xác thực OCR/eKYC, chuyển sang đã xác thực |
| Đại lý | Admin 11/100 được sửa; Admin đại lý 64 chỉ xem |
| O2O/Tự doanh | QL KD cung cấp dữ liệu, IT nhập CMS; Agent đọc dữ liệu phục vụ báo cáo |
| Tài khoản đã tồn tại | Chính sách cung cấp dữ liệu để IT import một lần |

Cần chốt (Q07/Q10/Q11):

- Hoàn thành OCR và eKYC là hoàn thành cả hai, thời điểm hệ thống nhận kết quả hay thời điểm nghiệp vụ? Duyệt thủ công được tính không?
- Xác thực lại có đổi ngày bắt đầu bán không? Đề xuất chỉ ghi lần đầu nếu chưa có giá trị, tránh reset ngoài ý muốn.
- “Ngày BC” trên CMS có đúng nghĩa không? Không tái sử dụng chỉ vì có cùng kiểu ngày.
- Có cho nhập ngày tương lai, xóa ngày hoặc sửa ngày khi đã đề xuất chấm dứt không?
- Khi sửa ngày, tái tính theo lịch sử chính sách/GD nào; có sửa lịch sử, gửi lại thông báo và thay đổi room không?
- Dữ liệu import thiếu hoặc sai mapping sale phải được báo lỗi, không mặc định ngày tạo account.

## 9. Dashboard, tìm kiếm và export — US-01/02/03

### 9.1. Cột hiển thị

- STT, họ tên sale, ID hệ thống, mã nhân viên, mã định danh, SĐT, email.
- Bộ phận KD trực thuộc, vùng quản lý, trạng thái hoạt động, ngày bắt đầu bán.
- Loại chu kỳ `x tháng - y tháng`.
- Chính thức gần nhất: ngày bắt đầu, ngày kết thúc, số ngày còn lại, số GD.
- Thử thách gần nhất: ngày bắt đầu, ngày kết thúc, số ngày còn lại, số GD.
- Ngày GD hợp lệ gần nhất và phân loại.

| Phân loại hiện tại | Nhóm chính thức | Nhóm thử thách |
| --- | --- | --- |
| Đang bán chính thức | Chu kỳ chính thức hiện tại | Dấu `-`, kể cả đã từng thử thách |
| Đang thử thách | Chu kỳ chính thức thất bại liền trước | Chu kỳ thử thách hiện tại |
| Đề xuất chấm dứt | Chu kỳ chính thức cuối cùng | Chu kỳ thử thách cuối cùng |

Chưa rõ “Ngày GD gần nhất” là toàn bộ lịch sử hay chỉ chu kỳ hiển thị. Với chính sách tuần tự, hai giai đoạn gần nhất có thể dùng phiên bản khác nhau; cần xác nhận cách hiển thị “Loại chu kỳ”.

### 9.2. Thao tác

- Đường dẫn menu: Quản lý sale → Theo dõi chu kỳ bán hàng.
- Pin cột STT/họ tên; mặc định họ tên A–Z; tối đa 20 dòng/trang.
- Search gần đúng theo họ tên, ID hệ thống, mã nhân viên, mã định danh.
- Sort theo họ tên, cột thời gian, số GD. Cần chỉ rõ từng cột, thứ tự null và quy tắc tên tiếng Việt.
- Filter nhiều lựa chọn theo bộ phận KD, trạng thái hoạt động và phân loại.
- Bộ lọc đại lý/vùng chủ quản chưa khớp mô hình miền/phòng/nhóm; cần tách quy tắc theo loại tổ chức.
- Mô tả US-02 nhắc filter thời gian/xếp hạng doanh số/GD nhưng bảng chi tiết không có. Không tự thêm các filter này trước khi PO xác nhận.

### 9.3. Excel

- Xuất toàn bộ kết quả search/filter được phép xem, không chỉ trang hiện tại.
- Cột thông tin tương ứng dashboard, định dạng Excel.
- Nguồn giới hạn 50.000 dòng và yêu cầu tự cắt khi vượt. Đề xuất thông báo rõ số dòng bị giới hạn và dùng sort ổn định để biết 50.000 dòng nào được xuất.
- Cần chốt template đúng tính năng; link hiện tại mang tên template xếp hạng sale.
- Backend áp dụng quyền tương tự API list; không nhận phạm vi export từ frontend rồi bỏ qua kiểm tra.

## 10. Thông báo — US-05

| Loại | Thời điểm theo nguồn | Nội dung chính |
| --- | --- | --- |
| Nhắc chính thức | Trước ngày kết thúc theo mốc/giờ cấu hình | Chu kỳ chính thức sắp kết thúc, nhắc đạt chỉ tiêu |
| Chính thức thất bại | Ngày sau kết thúc, giờ cấu hình | Đã chuyển qua thử thách do chưa đạt |
| Nhắc thử thách | Trước ngày kết thúc theo mốc/giờ cấu hình | Thử thách sắp kết thúc, nhắc đạt chỉ tiêu |
| Thử thách thất bại | Ngày sau kết thúc, giờ cấu hình | Thuộc diện xem xét chấm dứt hợp tác |

Kênh: web/app Agent. Người nhận: sale được theo dõi. Mốc nhắc vượt độ dài chu kỳ được đưa về ngày đầu chu kỳ theo SRS.

Thiếu quy tắc: sale chính thức đã đạt có còn nhận nhắc không; có thông báo thành công/thoát thử thách không; sale inactive có nhận không; GD đến trước giờ gửi có hủy thông báo thất bại không; nhiều mốc cùng rơi vào ngày đầu chu kỳ gửi một hay nhiều lần (Q14).

Đề xuất kỹ thuật:

- Mỗi thông báo có khóa chống trùng theo sale + chu kỳ + loại + mốc nhắc.
- Job chạy lại/restart không gửi lại cùng thông báo đã xử lý.
- Khi đổi lịch, tái tính hoặc đóng thử thách sớm, hủy thông báo chưa gửi không còn phù hợp.
- Có cơ chế retry và phục hồi khi job bỏ lỡ lịch; cần chốt cửa sổ gửi bù.

## 11. Tác động đến code hiện tại

Các điểm dưới đây được kiểm tra sơ bộ trong repo hiện tại; chưa phải kết luận toàn bộ hệ thống đã có/chưa có chức năng.

| Khu vực | File tham chiếu | Ý nghĩa khi triển khai |
| --- | --- | --- |
| Profile sale co-broke | `src/main/java/vn/vinhomes/cobroker/core/model/CoBrokerProfile.java` | Entity đang có accountId/coBrokerId, thông tin cá nhân, tổ chức và status; chưa có field ngày bắt đầu bán trong entity này |
| Trạng thái profile | `src/main/java/vn/vinhomes/cobroker/core/enums/CoBrokerProfileStatusEnum.java` | Có DRAFT/ACTIVE/INACTIVE/FROZEN/REQUEST_UPDATE; không khớp trực tiếp toàn bộ nhãn hoạt động trong SRS, phải làm mapping |
| Hoàn thành xác thực | `src/main/java/vn/vinhomes/cobroker/core/service/applicant/ApplicantEkycCompletionService.java` | Có luồng hoàn thành eKYC/activation; cần rà thêm OCR và manual approval để bắt đúng sự kiện hoàn thành lần đầu |
| Dữ liệu tính room | `src/main/java/vn/vinhomes/cobroker/core/repository/AgencyCobrokerRepository.java` | Có query đếm sale phục vụ module phân phối; cần xác định các query nào phải loại sale thử thách/đề xuất chấm dứt |
| Room | `src/main/java/vn/vinhomes/cobroker/core/service/distribution/impl/RoomServiceImpl.java` | Room có tính theo số sale hợp lệ; giảm số sale có thể ảnh hưởng ngân sách và phân bổ hiện có |
| Trigger tính lại room | `src/main/java/vn/vinhomes/cobroker/core/service/distribution/helpers/AgencyCobrokerRoomListener.java` và `AgencyRoomRecalcHook.java` cùng thư mục | Listener hiện phản ứng với thay đổi entity AgencyCobroker. Nếu chu kỳ lưu bảng riêng, cập nhật chu kỳ sẽ không tự kích hoạt listener này; phải tích hợp trigger rõ ràng |
| Điểm theo lực lượng sale | `src/main/java/vn/vinhomes/cobroker/core/service/distribution/AgencyScoreSaleCountSweeper.java` | Có cơ chế refresh saleCount và tính lại điểm; PO cần xác nhận thử thách có ảnh hưởng điểm lực lượng ngoài room không |
| Thông báo | `src/main/java/vn/vinhomes/cobroker/core/service/notification/outbox/NotificationOutboxService.java` | Có scheduled outbox, enqueue/cancel; có thể mở rộng nhưng cần định nghĩa loại, handler và khóa chống trùng cho chu kỳ |

**Không đồng nhất “không tính vào room” với “khóa tài khoản sale”.** Cần làm rõ có giảm room đại lý, chặn quyền lấy căn cá nhân, thu hồi căn đã phân bổ, xử lý room thủ công hay chỉ loại khỏi công thức cấp room mới (Q06). Không mặc định mọi mô hình room đều chịu cùng một tác động.

Phạm vi O2O/Tự doanh, nguồn SAP, CMS và quyền quản lý có thể nằm ở service khác. Phải xác định service sở hữu dữ liệu và hợp đồng tích hợp trước khi gom mọi phần vào cobroker-core.

## 12. Danh sách quyết định PO/BO cần chốt

| ID | Vấn đề | Câu hỏi cần trả lời | Mức ảnh hưởng |
| --- | --- | --- | --- |
| Q01 | Đạt giữa kỳ chính thức | Giữ đến cuối kỳ như phần nguyên tắc hay reset ngay? Xóa ghi chú BO confirm sau khi chốt | Chặn engine |
| Q02 | Chỉ tiêu thử thách > 1 | Thất bại khi GD < mục tiêu hay chỉ khi GD = 0? Một GD có đủ thoát thử thách không? | Chặn engine |
| Q03 | Luật mới | Có cho áp dụng tức thì không? Phạm vi reset, GD giữ lại, sale đã đề xuất chấm dứt, ranh giới áp dụng tuần tự? | Chặn cấu hình/engine |
| Q04 | Nguồn GD | TTĐC/TTKQ hay VBCN/HĐMB? ID GD, ngày nghiệp vụ, sale attribution, GD cá nhân hay đội? | Chặn SAP |
| Q05 | GD muộn/hủy | Có hồi tố/hoàn tác trạng thái và room không, xử lý tới mức nào? | Chặn tính đúng dữ liệu |
| Q06 | Room | Loại khỏi công thức nào? Căn đang giữ có thu hồi không? Tác động điểm lực lượng và O2O/Tự doanh? | Chặn room |
| Q07 | Sửa ngày bắt đầu | Tái tính lịch sử thế nào, có phục hồi trạng thái cuối, gửi lại thông báo không? | Chặn update/replay |
| Q08 | Quyền | Role vùng, scope đa role, role 100 toàn hệ thống, quyền export 25, quyền sửa/cấu hình? | Chặn API quyền |
| Q09 | Chuyển tổ chức/tái gia nhập | Giữ hay reset chu kỳ? Luật nào và ai được xem lịch sử? | Chặn mô hình định danh |
| Q10 | Ngày bắt đầu và thiếu dữ liệu | OCR+eKYC/manual lấy thời điểm nào; thiếu/future date hoặc thiếu policy hiển thị và tính room ra sao? | Chặn onboarding |
| Q11 | Dữ liệu trước go-live | Chạy lại từ ngày bắt đầu lịch sử hay bắt đầu mới? Có đủ GD/chính sách cũ; gửi cảnh báo/giảm room ngay không? | Chặn rollout |
| Q12 | Phân kỳ bàn giao | Cấu hình động tháng 10 hay tháng 11? Luật tạm trước UI cấu hình? | Chặn scope/estimate |
| Q13 | Lịch | Timezone, ngày cuối tháng/năm nhuận, số ngày còn lại, thời điểm chốt GD? | Chặn date logic |
| Q14 | Thông báo | Nhắc sale đã đạt/inactive, thông báo thành công, chống trùng mốc, hủy và gửi bù? | Chặn notification |
| Q15 | Giao diện/dữ liệu | Template export, filter thừa, mapping status, ngày GD gần nhất, loại chu kỳ khi khác phiên bản? | Chặn nghiệm thu UI |

Các lỗi cụ thể cần sửa trong tài liệu nguồn:

- Bảng “Phân loại” SRS đảo định nghĩa Đạt/Chưa đạt; phần nguyên tắc lại đúng theo so sánh chỉ tiêu.
- Ví dụ `31/11/2026` là ngày không tồn tại.
- Ví dụ 4 tháng bắt đầu 01/08 kết thúc 31/10 không khớp công thức; theo công thức là 30/11.
- PRD dùng “chấm dứt vĩnh viễn”, SRS dùng “đề xuất chấm dứt hợp tác”. Phải thống nhất tác động thực tế; chưa có cơ sở tự động chấm dứt/khóa account.
- US-02/US-03 và template còn nội dung từ tính năng xếp hạng; bảng thông báo US-05 đặt nội dung sai cột.

## 13. Đề xuất thiết kế và chia việc

Đây là định hướng kỹ thuật, không phải schema/API đã được duyệt.

### 13.1. Thành phần dữ liệu

1. **Sale được theo dõi:** định danh ổn định, nhóm, tổ chức, ngày bắt đầu bán và nguồn/người sửa ngày.
2. **Phiên bản chính sách:** ngày hiệu lực, cơ chế áp dụng, nhóm, số tháng/chỉ tiêu và lịch nhắc.
3. **Chu kỳ:** sale, loại, start/end dự kiến, ngày đóng thực tế nếu thoát sớm, policy version, kết quả và GD được tính.
4. **GD chuẩn hóa:** ID nguồn, sale, ngày nghiệp vụ, trạng thái và thông tin cập nhật để đối soát/replay.
5. **Projection hiện tại:** phân loại hiện tại và chu kỳ gần nhất dùng cho list/export/room.
6. **Audit/outbox:** thay đổi chính sách/ngày bắt đầu, chuyển trạng thái và tác vụ tích hợp/thông báo.

### 13.2. Luồng xử lý đề xuất

```text
Profile/CMS → ngày bắt đầu bán + đối tượng
SAP → giao dịch chuẩn hóa, chống trùng
Policy + ngày nghiệp vụ + giao dịch → tính/chuyển chu kỳ
Chuyển chu kỳ → cập nhật projection + audit + tác vụ room/thông báo
Projection + scope người dùng → dashboard / export
```

- Thiết kế engine tách luật nghiệp vụ khỏi scheduler và adapter SAP/CMS.
- Xử lý đồng thời cùng sale phải tránh tạo hai chu kỳ mới; bảo vệ bằng transaction/locking hoặc ràng buộc dữ liệu phù hợp.
- Job chạy bù qua nhiều ngày phải tạo đúng chuỗi chu kỳ, không chỉ chuyển một bước rồi bỏ mất thời gian.
- API GET dashboard không nên tự ghi chu kỳ và phát thông báo.
- Tái tính phải có cách xem ảnh hưởng trước khi cập nhật hàng loạt, nhất là room và trạng thái cuối.

### 13.3. Thứ tự triển khai

| Bước | Công việc | Điều kiện đầu vào |
| --- | --- | --- |
| 1 | Chốt quyết định nghiệp vụ, mapping định danh/quyền và service owner | Bảng Q01–Q15, mẫu payload thực tế |
| 2 | Chốt schema, API, cơ chế policy version và engine thời gian | Luật chu kỳ/date đã xác nhận |
| 3 | Ngày bắt đầu bán, import/backfill và dữ liệu policy ban đầu | Q07/Q10/Q11 |
| 4 | Adapter SAP, chống trùng và đối soát | Q04/Q05 |
| 5 | Engine chu kỳ, job catch-up, audit và projection | Q01/Q02/Q03/Q13 |
| 6 | Quyền, list/filter/export, UI cấu hình | Q08/Q12/Q15 |
| 7 | Room và thông báo, phục hồi khi tích hợp lỗi | Q06/Q14 |
| 8 | Dry-run dữ liệu cũ, UAT và rollout theo scope đã chốt | Tất cả quyết định ảnh hưởng production |

## 14. Tiêu chí nghiệm thu và kiểm thử

Các test phụ thuộc Qxx chỉ được coi là có expected result cuối cùng sau khi quyết định được chốt.

| ID | Tình huống | Kết quả cần kiểm tra |
| --- | --- | --- |
| AC01 | Chính thức đạt giữa kỳ | Đúng nhãn, ngày kết thúc không đổi nếu chọn luật giữ đến cuối kỳ |
| AC02 | Chính thức đạt đến cuối kỳ | Chu kỳ chính thức mới bắt đầu hôm sau, GD không cộng lại |
| AC03 | Chính thức chưa đạt đến cuối kỳ | Ngày kết thúc vẫn nhận GD; hôm sau mới thử thách và cập nhật room |
| AC04 | Thử thách đạt giữa kỳ | Đóng đúng ngày đạt; hôm sau chính thức mới; thông báo cũ được xử lý phù hợp |
| AC05 | Thử thách có 1 GD, mục tiêu 2 | Không bỏ sót trường hợp chưa đạt; expected theo Q02 |
| AC06 | Thử thách thất bại | Đề xuất chấm dứt, không tự sinh thêm chu kỳ và không tự khóa account |
| AC07 | Cuối tháng/năm nhuận | Đúng mốc 28/29/30/31 theo Q13 |
| AC08 | Policy mới giữa chính thức/thử thách | Đúng phiên bản, ranh giới tuần tự/tức thì và đối tượng áp dụng |
| AC09 | GD trùng, muộn, hủy hoặc đổi sale | Đúng số GD, không cộng đôi, hồi tố theo Q04/Q05 |
| AC10 | Hai worker/GD đồng thời, job chạy lại | Không tạo chu kỳ/thông báo trùng |
| AC11 | Job dừng nhiều ngày | Catch-up đúng chuỗi, không mất dữ liệu và gửi bù theo chính sách |
| AC12 | Sửa ngày bắt đầu, xác thực lại | Đúng quy tắc tái tính và không reset ngoài ý muốn |
| AC13 | Sale inactive/chuyển tổ chức/tái gia nhập | Chu kỳ, quyền và room đúng luật đã chốt |
| AC14 | User khác đại lý/vùng/team | List/detail/export/update không vượt scope; thử trực tiếp API |
| AC15 | Dashboard ba nhóm trạng thái | Chính thức/thử thách gần nhất hiển thị đúng; null/dấu `-` và số ngày đúng |
| AC16 | Export nhiều trang, >50.000 dòng | Đúng filter/scope/sort/cột; giới hạn có thông tin rõ ràng |
| AC17 | Mốc nhắc dài hơn chu kỳ hoặc trùng nhau | Đúng ngày đầu kỳ và số lần gửi đã chốt |
| AC18 | Tích hợp room lỗi/retry | Kết quả hội tụ, không thu hồi/cộng room hai lần; phân bổ cũ theo Q06 |
| AC19 | Import lịch sử, thiếu ngày hoặc thiếu policy | Không âm thầm mặc định; có báo lỗi/đối soát và expected theo Q10/Q11 |

Trước go-live cần đối soát mẫu do BO xác nhận: danh sách sale, GD nguồn, các chu kỳ suy ra, phân loại và room trước/sau. Không dùng UAT chỉ kiểm tra giao diện để thay cho đối soát dữ liệu.

## 15. Hướng dẫn thực hiện cho AI agent

1. Đọc file này và tài liệu nguồn; kiểm tra quyết định Qxx đã có cập nhật chưa.
2. Đọc `AGENTS.md` nếu repo có; kiểm tra branch và thay đổi local trước khi sửa.
3. Phân biệt yêu cầu nguồn, đề xuất thiết kế và câu hỏi chưa chốt. Không biến ví dụ/chính sách mẫu thành luật mặc định production.
4. Nếu nhiệm vụ phụ thuộc Qxx chưa có câu trả lời, ghi rõ phần bị chặn; vẫn có thể làm phần độc lập như hợp đồng API sơ bộ hoặc thiết kế có thể review. Không kích hoạt side effect room/khóa account theo suy đoán.
5. Rà toàn bộ đường OCR/eKYC/manual approval và query đếm sale liên quan, không sửa duy nhất một đường rồi coi là hoàn thành.
6. Giữ trạng thái hoạt động và trạng thái chu kỳ riêng. Không sửa `status` để mô phỏng thử thách.
7. Thêm test nghiệp vụ tại các ranh giới có rủi ro trong mục 14; chạy kiểm tra phù hợp và báo rõ phần chưa xác minh.
8. Khi bàn giao, nêu thay đổi, quyết định Qxx đã áp dụng, migration/backfill, kiểm chứng và phụ thuộc service khác còn lại.

**Điều kiện để bắt đầu triển khai engine production:** có quyết định rõ cho luật chuyển chu kỳ, nguồn và hồi tố GD, policy mới, thời gian và khởi tạo dữ liệu cũ. Tích hợp room chỉ kích hoạt sau khi xác nhận tác động và đối soát dữ liệu.
