# BDSKD-9533 — Phân tích yêu cầu PRD

Ngày phân tích: 05/10/2026.

Nguồn: `PRD-Theo dõi kết quả bán hàng - ngừng hợp tác.odt`, phiên bản 0.1, cập nhật 23/09/2026, trạng thái chờ phê duyệt.

Tài liệu này chỉ phân tích nội dung PRD được cung cấp trong thư mục. Không dùng SRS để bổ sung yêu cầu và chưa kiểm tra thiết kế, Excel hoặc Q&A được dẫn bên ngoài. Các nhận định và câu hỏi dưới đây không phải quyết định nghiệp vụ đã được PO/BO phê duyệt.

## 1. Yêu cầu tổng thể của PO

PO muốn xây dựng công cụ theo dõi hiệu quả bán hàng để quyết định tiếp tục cấp nguồn lực hay đưa sale vào diện xem xét ngừng hợp tác.

Đây là yêu cầu gồm báo cáo, chính sách chu kỳ, thông báo và tác động tới hạn mức căn (room). Màn hình thống kê là nơi thể hiện kết quả của các quy tắc nghiệp vụ đó.

## 2. Bài toán cần giải quyết

Hiện việc theo dõi sale không đạt hiệu quả đang làm thủ công, dẫn đến:

- Khó biết sale nào sắp hết thời gian bán nhưng chưa có giao dịch.
- Chậm phát hiện sale không hiệu quả để điều chỉnh hạn mức căn.
- Dễ tranh cãi khi quyết định ngừng hợp tác vì thiếu dữ liệu và quy tắc minh bạch.
- Khó vận hành chính sách khác nhau giữa Đại lý, O2O và Tự doanh khi chính sách thay đổi.

Kết quả mong muốn:

- Sale biết thời hạn và chỉ tiêu cần đạt.
- Quản lý biết tình trạng lực lượng bán hàng để đôn đốc.
- Khối Chính sách có căn cứ xác định sale cần xem xét ngừng hợp tác.
- Khối Chính sách chủ động điều chỉnh luật trên hệ thống.
- Nguồn lực và hạn mức căn được phân bổ dựa trên hiệu quả bán hàng.

PRD đề cập đo thời gian rà soát sale và tỷ lệ lỗi khi áp dụng chính sách mới, nhưng chưa có giá trị hiện tại, mức mục tiêu hoặc kỳ đo. Vì vậy chưa đủ để đánh giá mức độ thành công của sản phẩm bằng số liệu.

## 3. Người dùng và nhu cầu

| Người dùng | Nhu cầu trong PRD | Kết quả mong muốn |
| --- | --- | --- |
| Sale | Nhận nhắc nhở trước hạn và thông báo khi hết chu kỳ trên web/app Agent | Biết thời hạn KPI để chủ động bán hàng |
| Admin đại lý / Quản lý kinh doanh | Xem sale thuộc đội ngũ mình quản lý | Nhận diện và đôn đốc sale đang cảnh báo |
| Khối Chính sách / Kinh doanh | Xem toàn bộ báo cáo, lọc sale đề xuất ngừng hợp tác và cấu hình chính sách | Có dữ liệu và công cụ vận hành chính sách |

PRD xác định quyền theo vai trò nghiệp vụ, chưa có ma trận role kỹ thuật hoặc quy tắc xác định phạm vi tổ chức.

## 4. Mô hình nghiệp vụ cốt lõi

PRD chia quá trình bán hàng thành hai giai đoạn:

| Giai đoạn | Mục đích | Kết quả |
| --- | --- | --- |
| Chu kỳ bán chính thức | Cho sale một khoảng thời gian để đạt chỉ tiêu | Đạt thì tiếp tục chu kỳ bán; không đạt thì chuyển cảnh báo |
| Chu kỳ cảnh báo | Cho thêm cơ hội bán nhưng không được tính vào room | Bán được thì trở lại chính thức; hết hạn không bán được thì đề xuất chấm dứt |

Chính sách được tài liệu đề cập:

- Đại lý: 4 tháng bán chính thức, 6 tháng cảnh báo; ví dụ cấu hình chỉ tiêu 1 GD.
- O2O/Tự doanh: chu kỳ cơ bản 3 tháng bán chính thức, 3 tháng cảnh báo.

Các thông số này chưa có đủ thông tin về ngày hiệu lực và phê duyệt để coi là cấu hình production cuối cùng. PRD cũng chưa xác định rõ chỉ tiêu riêng của O2O/Tự doanh hoặc chỉ tiêu có khác nhau giữa hai giai đoạn hay không.

## 5. Luồng vận hành theo PRD

### Bước 1 — Xác định ngày bắt đầu bán

Sale hoàn thành OCR/eKYC. Hệ thống ghi nhận ngày bắt đầu bán và kích hoạt chu kỳ chính thức.

PRD yêu cầu có phương án nhập ngày bắt đầu bán thủ công cho trường hợp cần hỗ trợ. Chưa quy định người được sửa, giới hạn giá trị và tác động khi sửa ngày.

### Bước 2 — Thiết lập chính sách

Khối Chính sách cấu hình nhóm áp dụng, thời gian chu kỳ, số GD cần đạt và ngày hiệu lực. Hệ thống lưu cấu hình và chờ đến ngày hiệu lực để áp dụng.

### Bước 3 — Theo dõi hằng ngày

Hệ thống cập nhật số ngày còn lại và số giao dịch được ghi nhận từ SAP.

### Bước 4 — Kết thúc chu kỳ chính thức

- Nếu đạt chỉ tiêu: mở chu kỳ chính thức tiếp theo.
- Nếu không đạt: chuyển sang cảnh báo và loại khỏi điều kiện tính room.

PRD nghiêng về luật đạt giữa chu kỳ vẫn tiếp tục bán đến hết kỳ, không reset ngay. Tuy nhiên tài liệu còn ghi “cần BO confirm thêm”, nên chưa thể coi đây là quyết định cuối cùng.

### Bước 5 — Xử lý chu kỳ cảnh báo

- Có GD mới: kết thúc cảnh báo và mở chu kỳ chính thức mới từ ngày hôm sau.
- Hết hạn vẫn không có GD: gán nhãn đề xuất chấm dứt hợp tác.

Chưa rõ “có GD mới” nghĩa là có ít nhất 1 GD hay đạt đủ chỉ tiêu cấu hình. Khác biệt này quan trọng nếu chỉ tiêu lớn hơn 1.

## 6. Chức năng trong phạm vi

| Nhóm chức năng | Yêu cầu của PRD | Giá trị mang lại |
| --- | --- | --- |
| Dashboard | Theo dõi hai loại chu kỳ, số ngày còn lại, số GD và phân loại | Quản lý nhận diện sale cần đôn đốc |
| Phân loại tự động | Đạt yêu cầu, chưa đạt, cảnh báo, đề xuất chấm dứt | Thống nhất kết quả đánh giá |
| Bộ lọc | Theo nhóm/tổ chức, trạng thái hoạt động và phân loại | Khoanh vùng sale cần xử lý |
| Excel | Xuất thông tin đang theo dõi | Phục vụ báo cáo và xử lý nghiệp vụ |
| Cấu hình chính sách | Thời gian, số GD, nhóm áp dụng và ngày hiệu lực | Chính sách chủ động thay đổi luật |
| Thông báo | Nhắc trước hạn và thông báo kết quả chu kỳ | Sale biết thời hạn KPI |
| Ngày bắt đầu bán | Tự ghi từ xác thực, có phương án nhập tay | Có mốc bắt đầu tính chu kỳ |
| Tích hợp room | Sale cảnh báo không được tính vào hạn mức căn | Điều chỉnh nguồn lực theo hiệu quả |

PRD có nội dung thông báo ở phần tóm tắt, hành trình và mô tả phạm vi, nhưng chưa đặc tả đầy đủ lịch gửi, điều kiện gửi và nội dung thông báo.

## 7. Ngoài phạm vi

Theo PRD:

- Reset chu kỳ thủ công cho từng sale.
- Màn hình hiển thị toàn bộ lịch sử chu kỳ; IT xuất dữ liệu khi có yêu cầu.

Cần phân biệt sửa ngày bắt đầu bán với reset chu kỳ thủ công. Nếu sửa ngày dẫn đến tính lại toàn bộ chu kỳ, thao tác đó có thể tạo tác động tương tự reset. PRD chưa giải quyết ranh giới này.

## 8. Nguyên tắc nghiệp vụ đã thể hiện

| Nguyên tắc | Nội dung trong PRD | Mức độ rõ ràng |
| --- | --- | --- |
| Nhóm áp dụng | Đại lý, O2O, Tự doanh có chính sách riêng | Đã xác định nhóm, chưa xác định mapping dữ liệu |
| Đối tượng độc quyền | Một nhóm chỉ nằm trong một khối của cùng cấu hình | Khá rõ ở mức thao tác cấu hình |
| Ngày hiệu lực | Lưu cấu hình và áp dụng từ ngày hiệu lực | Chưa mô tả giới hạn ngày và xung đột cấu hình |
| Chuyển luật | BR-03: hoàn thành chu kỳ đang chạy theo luật cũ rồi dùng luật mới | Chưa rõ ranh giới một giai đoạn hay cả cặp chu kỳ |
| Quyền xem | Chính sách xem toàn bộ; quản lý xem bộ phận mình | Chưa có ma trận role và nguồn scope |
| Cảnh báo | Sale tiếp tục bán nhưng không được tính vào room | Chưa rõ hành động room cụ thể |
| Thoát cảnh báo | Chính thức mới bắt đầu từ ngày hôm sau | Chưa rõ ngưỡng GD và giờ chuyển |

## 9. Dữ liệu và hệ thống phụ thuộc

| Dữ liệu / Tác động | Hệ thống được PRD đề cập | Điều cần xác nhận |
| --- | --- | --- |
| GD bán hàng | SAP | Trạng thái hợp lệ, khóa GD, ngày ghi nhận và định danh sale |
| Xác thực và trạng thái tài khoản | Agent CRM | Thời điểm hoàn thành, xử lý xác thực lại và dữ liệu cũ |
| Room / Hạn mức căn | Core Giỏ hàng | Quy tắc giảm/khôi phục room, quyền lấy căn và căn đã phân bổ |
| Phân quyền quản lý | Chưa mô tả đầy đủ nguồn dữ liệu | Mapping vai trò và tổ chức quản lý |
| Ngày bắt đầu bán O2O/Tự doanh | Chưa mô tả riêng | Nguồn ngày phù hợp cho từng nhóm |

PRD đã nhận diện việc tích hợp Giỏ hàng là phụ thuộc có ảnh hưởng cao. Cần có đầu mối và phạm vi tích hợp rõ trước khi cam kết bàn giao toàn bộ tính năng.

## 10. Điểm chưa rõ hoặc chưa nhất quán

| ID | Vấn đề | Nội dung cần PO/BO xác nhận | Ảnh hưởng |
| --- | --- | --- | --- |
| P01 | Giao dịch hợp lệ | Luồng nói xác nhận TTKQ/ĐC; mục 8.1 nói ký VBCN/HĐMB. Mốc nào dùng để tính GD? | Có thể đếm sai toàn bộ chỉ tiêu |
| P02 | Đạt giữa kỳ chính thức | Giữ đến cuối kỳ hay reset ngay? Tài liệu vẫn còn ghi chú BO confirm | Ảnh hưởng mốc mọi chu kỳ tiếp theo |
| P03 | Ngưỡng thoát cảnh báo | Có 1 GD là đủ hay phải đạt chỉ tiêu cấu hình? | Không xử lý rõ chỉ tiêu lớn hơn 1 |
| P04 | Kết quả cuối | Chỉ gán nhãn đề xuất hay thực hiện chấm dứt? BR-02 có chữ “chấm dứt vĩnh viễn” | Có thể dẫn đến hành động vượt phạm vi |
| P05 | Tác động room | Giảm room đại lý, chặn quyền lấy căn cá nhân, hay cả hai? Căn đã phân bổ có bị thu hồi không? | Ảnh hưởng bán hàng và nguồn lực hiện tại |
| P06 | Ranh giới chuyển chính sách | Hoàn tất một giai đoạn hay cả cặp chính thức + cảnh báo? | Có thể dùng sai luật khi chuyển giai đoạn |
| P07 | Ngày bắt đầu bán | OCR/eKYC áp dụng thế nào cho từng nhóm? Hoàn thành cả hai hay một trong hai? | Chưa có mốc thống nhất |
| P08 | Sửa ngày bắt đầu | Ai được sửa? Có tính lại lịch sử và room không? | Có thể tạo reset gián tiếp |
| P09 | Dữ liệu trước go-live | Tính từ lịch sử hay mở chu kỳ mới khi go-live? Có đủ dữ liệu GD/chính sách cũ không? | Chưa xác định được trạng thái ban đầu |
| P10 | GD hủy/đến muộn | Có thu hồi chỉ tiêu và hoàn tác kết quả chu kỳ không? | Kết quả có thể đổi sau khi đã chốt |
| P11 | Chuyển tổ chức/tái gia nhập | Giữ hay reset chu kỳ? Chính sách nhóm nào áp dụng? | Có thể sai policy và scope |
| P12 | Thông báo | Lịch, nội dung, người nhận, điều kiện gửi, sale đã đạt có nhận nhắc không? | Chưa đủ để nghiệm thu |
| P13 | Độ cập nhật | Dashboard gọi thời gian thực nhưng xử lý mô tả job hằng ngày. Độ trễ chấp nhận là bao lâu? | Không rõ kỳ vọng vận hành |
| P14 | Thời gian chu kỳ | Cách cộng tháng, ngày cuối tháng, ngày kết thúc và timezone? | Dễ sai ở ranh giới ngày |
| P15 | Phân quyền | Role kỹ thuật, phạm vi quản lý, người nhiều role và quyền export/cấu hình? | Chưa đủ thiết kế quyền truy cập |
| P16 | Cấu hình | Có được sửa/hủy cấu hình? Nhiều cấu hình cùng ngày xử lý thế nào? | Chưa đủ vận hành chính sách |
| P17 | Giao dịch của quản lý | Tính GD cá nhân hay GD của đội/người thuộc quyền? | Có thể sai đánh giá sale các cấp |

**PRD chưa đủ căn cứ để dev tự động khóa tài khoản hoặc chấm dứt hợp đồng.** PO cần xác nhận hành động cuối cùng và hệ thống chịu trách nhiệm.

## 11. Kế hoạch bàn giao

| Mốc | Thời gian trong PRD |
| --- | --- |
| UAT | 25–30/10/2026 |
| Go-live chính | 31/10/2026 |
| Công cụ cấu hình động | Muộn nhất 30/11/2026 |

Cần chốt phạm vi từng đợt. Nếu dashboard và xử lý chu kỳ chạy tháng 10 nhưng UI cấu hình chưa có, phải xác định:

- Chính sách ban đầu được nhập bằng cách nào và ai phê duyệt.
- Ai được thay đổi chính sách trước khi có UI.
- Cách lưu ngày hiệu lực và lịch sử thay đổi.
- Những chức năng cấu hình nào được làm trong tháng 10, phần nào sang tháng 11.

PRD có mốc phát triển/tích hợp là TBU, nên chưa đủ cơ sở đánh giá tính khả thi của lịch bàn giao chỉ từ tài liệu này.

## 12. Đánh giá tiêu chí nghiệm thu trong PRD

PRD nêu ba nhóm tiêu chí:

1. Đếm ngược và chuyển từ bán chính thức sang cảnh báo đúng.
2. Sale cảnh báo phát sinh GD thì trở lại bán chính thức.
3. Cấu hình lưu đúng ngày hiệu lực và cơ chế áp dụng.

Các tiêu chí này phản ánh đúng luồng chính nhưng chưa đủ bao phủ phạm vi. Đề xuất bổ sung sau khi chốt nghiệp vụ:

| Nhóm nghiệm thu | Kết quả cần mô tả rõ |
| --- | --- |
| Dashboard | Cột dữ liệu, nhãn phân loại và số GD đúng dữ liệu nguồn |
| Phân quyền | User chỉ xem/xuất dữ liệu thuộc phạm vi được phép |
| Bộ lọc / Excel | Kết quả lọc và nội dung export khớp nhau |
| Thông báo | Đúng người, điều kiện, ngày/giờ và nội dung |
| Room | Loại/khôi phục sale đúng thời điểm; xử lý căn đang phân bổ theo quyết định BO |
| Dữ liệu cũ | Kết quả khởi tạo được đối soát với mẫu BO xác nhận |
| Chính sách mới | Sale đang chạy chu kỳ không bị áp dụng sai phiên bản |
| Tình huống ngoại lệ | GD đến muộn/hủy, sửa ngày, chuyển nhóm và thiếu dữ liệu có kết quả xác định |

Yêu cầu “chuẩn xác 100%” cần được chuyển thành bộ ví dụ đầu vào và kết quả mong đợi cụ thể để dev/QA kiểm chứng.

## 13. Thứ tự chốt yêu cầu đề xuất

Trước khi viết đặc tả triển khai chi tiết, ưu tiên năm nhóm quyết định:

1. **Giao dịch:** mốc SAP, ngày ghi nhận, định danh sale, GD của cá nhân/đội và luật hủy/đến muộn.
2. **Chu kỳ:** đạt giữa kỳ, ngưỡng thoát cảnh báo, thời điểm chuyển và cách tính tháng.
3. **Kết quả cuối:** đề xuất chấm dứt hay hành động tự động, có khả năng phục hồi không.
4. **Room:** công thức/quyền chịu ảnh hưởng, căn đang giữ và thời điểm giảm/khôi phục.
5. **Khởi tạo và chính sách:** dữ liệu cũ, ngày bắt đầu bán, sửa ngày và ranh giới áp dụng luật mới.

Sau đó chốt quyền, dashboard, export, thông báo và phạm vi bàn giao từng đợt.

## 14. Kết luận review PRD

PRD đã mô tả được mục tiêu sản phẩm, đối tượng, luồng nghiệp vụ chính và phạm vi dự kiến. Có thể dùng để phân rã công việc và xác định phụ thuộc giữa dữ liệu sale, SAP, xử lý chu kỳ, chính sách, dashboard, thông báo và room.

PRD chưa đủ chi tiết để chốt logic production. Dev và AI agent cần sử dụng danh sách P01–P17 để thu thập quyết định nghiệp vụ, không tự điền các khoảng trống bằng giả định. Bước tiếp theo là chốt các quyết định này, rồi đối chiếu SRS và xây dựng hướng dẫn triển khai.
