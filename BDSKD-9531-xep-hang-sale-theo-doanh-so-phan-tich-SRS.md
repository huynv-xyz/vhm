# BDSKD-9531 — Phân tích SRS xếp hạng sale theo doanh số

Cập nhật 05/10/2026. Nguồn chính: [SRS local](<SRS - Xếp hạng Sale theo doanh số.md>), Under approve, target 31/10/2026. [Trang Confluence PO cung cấp](https://vin3s.atlassian.net/wiki/spaces/BMAS/pages/3174598094) chưa đọc được do Atlassian yêu cầu kết nối lại; không khẳng định bản local là phiên bản mới nhất. [PRD](PRD_Xep_hang_Sale.odt) dùng để đối chiếu, không tự ghi đè đặc tả SRS.

Kiến trúc kế thừa định hướng bộ 9533: service/DB riêng, profile-mw chỉ đọc, GD nhận Kafka sale-pipeline; không phụ thuộc core-broker. Đây là quyết định triển khai, không phải nội dung nguyên văn SRS.

## 1. Sáu US cần bàn giao

| US | Kết quả cần có |
| --- | --- |
| US-01 | Bảng số liệu/hạng doanh số theo sale và quý |
| US-02 | Search/sort/filter trong phạm vi quyền |
| US-03 | Excel toàn tập đã lọc, tối đa 50.000 dòng |
| US-04 | Cấu hình và lịch sử tỷ lệ/ngưỡng/quota theo quý và đối tượng |
| US-05 | Badge hạng trên avatar web/app |
| US-06 | Sale tải chứng nhận PNG/JPEG |

US-05 và US-06 nằm chung một mục trong SRS; vẫn là hai tính năng để triển khai/kiểm thử riêng. Không thay bằng thông báo và import ngày bán của 9533.

## 2. Sale nào được tính, ai được xem?

### Đối tượng

| Nhóm | Điều kiện SRS |
| --- | --- |
| Đại lý | Thuộc đại lý, role sale member 21 |
| O2O | Thuộc khối O2O, role 21/20/23/60 |
| Tự doanh | Thuộc khối Tự doanh, role 21/20/23/60 |

SRS dẫn root O2O 1060/Tự doanh 1132 trên CMS staging; mapping theo môi trường phải xác nhận. Sale được xếp hạng không phụ thuộc trạng thái active/inactive. Không lọc sale thử thách theo 9533 khỏi tập xếp hạng.

### Quyền

| Người gọi | Xem | Export / cấu hình |
| --- | --- | --- |
| 11/100 | Toàn bộ | Export; cấu hình |
| 64 | Sale của đại lý mình | Export; không cấu hình |
| Quản lý vùng chủ quản đại lý | Sale thuộc đại lý quản lý | Export; mã role/scope chưa chốt |
| 60/23/20 | Sale trong miền/phòng/nhóm quản lý | Export; không cấu hình |
| Chỉ role 21 | Bảng của bộ phận đang trực thuộc | Không export; không cấu hình |

Role 21 không chỉ xem riêng bản thân trên bảng; tải certificate và badge cá nhân vẫn cần xác minh chính chủ. Nếu nhiều role thì xác định quyền theo identity/scope được tin cậy, không chỉ kiểm tra có role 21 rồi chặn mọi trường hợp. Role 100 và quyền USER_EXPORTER 25 còn ghi chú cần kiểm tra; không tự coi 25 là bắt buộc.

## 3. Giao dịch và doanh số được ghi nhận thế nào?

| Nội dung | Luật SRS |
| --- | --- |
| Loại GD | Mua sơ cấp, sale được gắn là nhân sự phụ trách |
| Điều kiện | Đi tới bước KH xác nhận HĐMB/VBCN |
| Số GD | Một GD nghiệp vụ hợp lệ được đếm một lần |
| Doanh số | Giá trị giao dịch không VAT, không KPBT |
| Hủy sau xác nhận | Vẫn giữ ghi nhận, không quan tâm thành công hay hủy về sau |
| Phân quý | Theo mốc nghiệp vụ được thống nhất; không theo ngày nhận Kafka |

Nguồn triển khai đề xuất là pipeline Kafka theo hướng đã chốt ở 9533. Event phải có **mốc và giá trị riêng phù hợp 9531**. Event chỉ xác nhận TTĐC/TTKQ hoặc chỉ có count theo sale không đủ để tính doanh số 9531.

Không tính hai lần khi GD đi qua cả HĐMB và VBCN; cần ID nghiệp vụ chuẩn hóa và luật chọn mốc với pipeline. Hủy thông thường khác với correction vì ghi nhầm sale/giá/ngày hoặc ghi nhận nhầm mốc; quyền/luật correction sau chốt còn phải xác nhận.

## 4. Quý, điểm và xếp hạng

### Quý báo cáo

Quý là quý dương lịch. Báo cáo bắt đầu Q3/2026. Mặc định chọn quý trước quý hiện tại; được chọn tối đa 4 quý, cột quý cũ sang mới từ trái qua phải. SRS cho chọn quý hiện tại, nhưng chỉ có hạng chính thức sau khi quý kết thúc.

Để tính Q3/2026 cần doanh số Q2/2026. Q2 là dữ liệu đầu vào, không tự trở thành quý báo cáo được mở trên UI. Sale thực sự chưa có GD Q2 có thể dùng 0; nguồn chưa backfill Q2 không được xem là 0.

### Điểm doanh số

```text
score(Q) = weight_previous × revenue(Q−1) + weight_current × revenue(Q)
```

Ví dụ: quý trước 10 tỷ, quý này 20 tỷ, tỷ lệ 30/70 → điểm = 17 tỷ. BE lưu kết quả gốc để so sánh; UI làm tròn hai chữ số thập phân. Không dùng giá trị đã làm tròn để xét đồng điểm hoặc hạng.

Số GD và doanh số quý là chỉ số gốc của đúng quý. Điểm là kết quả weighted hai quý. Tỷ lệ 30/70 là ví dụ, không tự seed cấu hình production khi chưa có policy được duyệt.

### Điều kiện hạng

Kim cương/Bạch kim/Vàng có tên cố định. Mỗi hạng phải thỏa ngưỡng tối thiểu và quota/top. Không đủ điều kiện thì Không xếp hạng sau khi đã chốt; đang chờ dữ liệu/chốt quý phải hiển thị trạng thái riêng.

SRS nêu Kim cương 100 đầu tiên, Bạch kim 200 tiếp theo, Vàng 300 tiếp theo; 120 sale đủ ngưỡng Kim cương thì chỉ 100 được hạng, 20 còn lại xét Bạch kim. Đây là ví dụ, không là quota production mặc định.

Ghi chú tại biên quota cho phép hai sale đồng điểm cùng Kim cương. Cần PO xác nhận phạm vi áp dụng mọi hạng, quota có thể vượt bao nhiêu và cách dịch chuyển tầng tiếp. Sort A–Z/ID để ổn định phân trang không được dùng phá hòa điểm nghiệp vụ.

Phạm vi cạnh tranh phải cố định trước tính hạng; filter đại lý/phòng/dự án/active chỉ thay danh sách đọc, không xét lại quota trong tập nhỏ vừa lọc. SRS cấu hình theo audience nhưng chưa nói đủ rõ cạnh tranh chung ba nhóm hay từng audience.

## 5. Báo cáo và filter — US-01/02

Mỗi dòng sale gồm họ tên, Agent ID, mã nhân viên hoặc mã định danh, điện thoại/email, bộ phận, vùng quản lý, trạng thái hoạt động. Mỗi quý có số GD, doanh số, điểm doanh số và hạng. UI giữ cố định STT/họ tên; tối đa 20 dòng/trang; mặc định tên A–Z.

Search gần đúng họ tên/Agent ID/mã nhân viên/mã định danh. Sort số GD/doanh số/điểm tăng hoặc giảm cần chỉ rõ quý khi chọn nhiều quý. Filter bộ phận và trạng thái trong scope; trạng thái mặc định tất cả.

Filter hạng khi chọn nhiều quý phải chốt: khớp bất kỳ quý hay tất cả quý hoặc một quý cụ thể. Không tự áp hạng của quý mới nhất khi UI không chỉ rõ.

Filter dự án được SRS ghi tùy mức độ triển khai: chỉ số GD/doanh số thay theo dự án, **điểm/hạng và các cột còn lại giữ nguyên**. Có filter này không tạo hạng theo dự án. Nguồn danh sách dự án CMS cần được cấp; nếu chỉ thấy project_ids trong profile thì chưa chứng minh có tên/danh mục dự án. Chốt phạm vi trước UI/API.

## 6. Export — US-03

Xuất đầy đủ các cột/quý đang xem và toàn tập đã lọc trong scope, không chỉ trang hiện tại. File Excel tối đa 50.000 dòng; nếu vượt, SRS yêu cầu tự cắt danh sách. Giữ sort ổn định và cho biết total/exported/truncated; không thay bằng từ chối xuất như một nghiệp vụ tự thêm.

Chỉ role 21 không được xuất. Quyền role 25 cần xác nhận. Template Excel được dẫn link nhưng chưa đọc được; schema cột của file cần đối chiếu template khi có nội dung. Log xuất là đề xuất vận hành, không phải US riêng.

## 7. Cấu hình và lịch sử — US-04

- Chọn quý hiện tại/tương lai, một hoặc nhiều quý. UI không chỉnh quý đã qua; backdated của quý cũ là luồng vận hành riêng có kiểm soát/audit.
- Chọn một hoặc nhiều audience Đại lý/O2O/Tự doanh; trong một lần lưu, một audience chỉ ở một box.
- Mỗi box có tỷ lệ quý trước/quý này và ngưỡng/quota ba hạng. Tên ba hạng không động.
- Xem lịch sử danh sách, xem chi tiết read-only của bản đã lưu.

Cần chốt validation tỷ lệ tổng 100%, quota có được bỏ trống hay bằng 0, ngưỡng thứ tự và khi đã có cấu hình cùng audience/quý thì cập nhật bằng phiên bản mới hay chặn. Không cho sửa trực tiếp bản policy mà kết quả chốt đang tham chiếu.

## 8. Badge và chứng nhận — US-05/06

Trong quý Q, badge/cert trên avatar cá nhân dùng kết quả chính thức Q−1. Đầu quý mới reset sang quý trước mới, không giữ badge của Q−2 chỉ vì Q−1 đang xử lý. Chưa có kết quả Q−1 hiển thị chờ chốt, không giả định sale không đạt hạng.

Sale đạt Kim cương/Bạch kim/Vàng có nút tải PNG/JPEG. Web và app lấy cùng kết quả; certificate gắn sale/quý/hạng/template version. File được cấp theo quyền chính chủ, không tin Agent ID tùy ý trong request.

SRS scope có sale đại lý nhưng actor US-05/06 liệt kê 60/23/20/21; role 21 có thể bao phủ sale đại lý, cần mapping rõ. Template chính thức chờ design 5–7/10 trong bản nguồn; chưa có file local để xác minh.

## 9. Những điểm cần chốt trước bật chức năng phụ thuộc

| Điểm | Chặn phần nào? |
| --- | --- |
| Event milestone HĐMB/VBCN, net revenue, attribution, correction/backfill | Ledger và độ đúng score |
| Mapping người ổn định khi đổi Agent ID/đại lý/audience | Cộng điểm xuyên quý |
| Scope quota chung/từng audience, ngưỡng score/doanh số, ties/quota tầng tiếp | Chốt hạng chính thức |
| Cửa sổ T−1, source completeness, sửa quý đã chốt | Close/rebuild và cấp certificate |
| Điểm/hạng số lượng; filter dự án và scope quyền vùng | API/UI tương ứng |
| Sort/filter đa quý; quota/tỷ lệ validation và policy replacement | List/policy |
| Certificate template/nội dung/retention và reset badge chưa có kết quả | Badge/certificate |
| Mốc go-live công cụ cấu hình 31/10 hay 30/11 | Kế hoạch nghiệm thu |

Có thể implement adapter/ledger và tính chỉ số trước. Các luật xếp hạng chưa rõ không được tự mặc định rồi phát hành badge/chứng nhận trên dữ liệu thật.
