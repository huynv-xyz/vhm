# SRS \- Xếp hạng Sale theo doanh số

| **Product Overview** |   |
| --- | --- |
| Target date | 31/10/2026 |
| Epic Link | [\[BDSKD-9531\] Xếp hạng Sales theo doanh số/ số GD - Jira](https://vin3s.atlassian.net/browse/BDSKD-9531) |
| Document Status | Under approve |
| Người soạn | Hoàng Thị Mỹ Hạnh  |
| Design | Bản thiết kế thô: 1. Bảng xếp hạng: [https://vingroupjsc.sharepoint.com/:u:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20Sale%20theo%20doanh%20s%E1%BB%91/Demo\_xep\_hang\_sale.html?d=wba8e2f39c40c4bc8b0110eb1bedf1f9d&csf=1&web=1&e=ZKZHVr](https://vingroupjsc.sharepoint.com/:u:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20Sale%20theo%20doanh%20s%E1%BB%91/Demo_xep_hang_sale.html?d=wba8e2f39c40c4bc8b0110eb1bedf1f9d&csf=1&web=1&e=ZKZHVr) 2. Phiếu chứng nhận hạng sale: [Phiếu chứng nhận.pn](https://vingroupjsc.sharepoint.com/:i:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20Sale%20theo%20doanh%20s%E1%BB%91/Phi%E1%BA%BFu%20ch%E1%BB%A9ng%20nh%E1%BA%ADn.png?d=wdbfa86acfad94f41af27952a99a943de&csf=1&web=1&e=pAlwaq) |
| Scope | - Xây dựng bảng xếp hạng Sale theo doanh số GD hàng quý + công cụ cấu hình xếp hạng động - Gắn huy hiệu xếp hạng vinh danh và cho phép Sale tải phiếu chứng nhận hạng  Lưu ý: Áp dụng cho toàn bộ sale Đại lý (luồng co-broke) và sale các cấp của KD tự doanh/ O2O |
| Tài liệu liên quan | - PRD: [https://vingroupjsc.sharepoint.com/:w:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20Sale%20theo%20doanh%20s%E1%BB%91/PRD\_Xep\_hang\_Sale.docx?d=wcb5b4c525fdd46b9a86388c2229d2c2f&csf=1&web=1&e=pWMRG0](https://vingroupjsc.sharepoint.com/:w:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20Sale%20theo%20doanh%20s%E1%BB%91/PRD_Xep_hang_Sale.docx?d=wcb5b4c525fdd46b9a86388c2229d2c2f&csf=1&web=1&e=pWMRG0) - Tài liệu trao đổi khác: [Xếp hạng sale + cảnh báo dừng hợp tác.xlsx](https://vingroupjsc.sharepoint.com/:x:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20sale%20+%20c%E1%BA%A3nh%20b%C3%A1o%20d%E1%BB%ABng%20h%E1%BB%A3p%20t%C3%A1c.xlsx?d=wc4babf0c5ae14047b1fa6db34a15783d&csf=1&web=1&e=GTg9k6&nav=MTVfe0Q2RkU4NzQzLTMyMDEtNDkwMC04QjA4LTE5RDAzN0E2NjgxRX0) |

# US-1: \[Web\] Xem bảng xếp hạng Sale theo doanh số hàng quý

## Tổng quan:

| **Mục** | **Nội dung** |
| --- | --- |
| US ID | US-01 |
| Tên | \[Web\] Xem bảng xếp hạng Sale theo doanh số hàng quý |
| Actor | - Admin hệ thống (11)/ Quản lý đại lý (100): Xem bảng xếp hạng của toàn bộ Sale Vinhomes Check dev dùng tái sử dụng Quản lý đại lý (100) không hay thêm role mới - Admin đại lý (64): Xem bảng xếp hạng của Sale thuộc Đại lý - Quản lý Vùng chủ quản đại lý: Xem bảng xếp hạng của Sale thuộc các Đại lý quản lý (Check với c Huyền các xác định role này) - Giám đốc miền KD (60): Xem bảng xếp hạng của Sale thuộc miền KD quản lý  - Giám đốc phòng Kinh doanh (23): Xem bảng xếp hạng của Sale thuộc phòng KD quản lý - Trưởng nhóm Kinh doanh (20): Xem bảng xếp hạng của Sale thuộc nhóm KD quản lý - Chuyên viên Kinh doanh (21): Xem bảng xếp hạng của bộ phận sale đang trực thuộc |
| Mô tả | Xây dựng bảng xếp hạng doanh số cho sale theo quý hiển thị đẩy đủ:  - Thông tin của sale: Họ tên, ID… - Số lượng, doanh thu, điểm xếp hạng, hạng Kim cương, Bạch kim, Vàng theo quýLưu ý:  - Xếp hạng được tính vào thời điểm kết thúc ngày cuối cùng của quý → do đó qua quý sau mới có xếp hạng quý trước đó - Sale được xếp hạng không quan tâm tới trạng thái tài khoản Check lại BO báo cáo thực tế lấy từ thời điểm nào =\> Lấy từ Quý 3-2026 |
| Trigger | Truy cập module “**Quản lý sale**” chọn mục “**Bảng xếp hạng sale**” để bắt đầu sử dụng |
| Pre-condition | Được phân quyền và đăng nhập thành công trên hệ thống Agent |
| Post-condition | Hiển thị báo cáo theo đúng phạm vi được cho phép |

## ** Mô tả dữ liệu:**

| **STT** | **Nhóm thông tin** | **Tên cột** | **Định nghia** | **Công thức** | **Bắt buộc** | **Ví dụ** |
| --- | --- | --- | --- | --- | --- | --- |
|  | Thông tin sale | Họ tên sale | Danh sách toàn bộ sale gồm: 1.  Sale đại lý: Thuộc 1 đại lý cụ thể + role Sale member (21) 2.  Sale O2O, Tự doanh: Thuộc Khối O2O ([1060](https://stag-market-cms.vinhomes.vn/portal/team/1060)) hoặc Khối Tự doanh ([1132](https://stag-market-cms.vinhomes.vn/portal/team/1132)) + role: sale\_member (21); sale\_manager(20); sale\_director(23); regional\_director (60) |  | **x** | Hoàng Thị My Hạnh |
|  | ID hệ thống | ID duy nhất trên hệ thống Agent để định danh tài khoản sale đó |  | **x** | 17688534\_1848 |  |
|  | Mã nhân viên | - Mã nhân viên đối với sale O2O và Tự doanh  - Lấy từ profile nhân viên trên CMS |  |  | 3751459 |  |
|  | Mã định danh | - Mã định danh đối với sale Đại lý - Lấy từ profile sale trên VHM Agent |  |  | 036395729500 |  |
|  | Số điện thoại | - Đối với Sale đại lý: Lấy từ thông tin profile trên Agent - Đối với Sale O2O, Tự doanh: Lấy từ CMS |  |  |  |  |
|  | Email | - Đối với Sale đại lý: Lấy từ thông tin profile trên Agent - Đối với Sale O2O, Tự doanh: Lấy từ CMS |  |  |  |  |
|  | Bộ phận KD | Bộ phận sale đang trực thuộc trực tiếp |  | **x** | Nhóm O2O.02 |  |
|  | Vùng quản lý | - Đối với Đại lý: Là vùng chủ quản đại lý - Đối với O2O/ Tự doanh: Là vùng quản lý BP sale trực thuộc trên CMS org chart |  | **x** | Vùng O2O |  |
|  | Trạng thái hoạt động | Trạng thái hoạt động của sale: - Đang hoạt động - Ngưng hoạt động - Chờ xác thực - Nháp Lấy từ profile sale trên VHM Agent |  | **x** | Đang hoạt động |  |
|  | Các chỉ số Giao dịch theo Quý | Số GD | - Số giao dịch mua sơ cấp phát sinh trong Quý báo cáo mà sale được gắn là nhân sự phụ trách  - Được tính khi giao dịch đi tới bước **KH xác nhận HĐMB/VBCN **(không quan tâm sau đó giao dịch thành công hay bị hủy) |  | **x ** | 3 |
|  | Doanh số (vnđ) | - Tổng giá trị giao dịch mua sơ cấp (KHÔNG VAT KHÔNG KPBT) phát sinh trong Quý báo cáo mà sale được gắn là nhân sự phụ trách - Được tính khi giao dịch đi tới bước **KH xác nhận HĐMB/VBCN **(không quan tâm sau đó giao dịch thành công hay bị hủy) |  | **x** | 12.434.849.248 |  |
|  | Điểm theo doanh số | - Chấm điểm theo giá trị giao dịch cộng dồn từ quý trước và quý đang xếp hạng - Hiển thị trên UI làm tròn tới thập phân thứ 2 (Lưu ý: Dưới BE cần lưu kết quả gốc không làm tròn để xếp hạng) | = **x**% giá trị giao dịch quý trước + **y** % giá trị giao dịch quý đang xếp hạng(tham số x; y được cấu hình động) |  | 7.434.849.248,54 |  |
|  | Xếp hạng theo doanh số | Xếp hạng theo điểm doanh số. Gồm 3 hạng: - Kinh cương - Bạch kim - Vàng Chỉ hiện thị xếp hạng khi quý kết thúc, quý sau hiển thị xếp hạng của quý trước | Mỗi hạng đáp ứng đồng thời 2 điều kiện:  -Giá trị giao dịch tối thiểu = a-Lấy top từ  m đến n(tham số a; m; n được cấu hình động) |  | Bạch kim |  |

## ** Mô tả thiết kế:**

- Site map: Truy cập module “**Quản lý sale**” chọn mục “**Bảng xếp hạng sale**” để bắt đầu sử dụng
- Pin/Freezing cột STT và Họ tên sale
- Mặc định lọc Thời gian lọc quý trước đó của quý hiện tại (vd: hiện là 28/9 thuộc quý 3 → mặc định hiển thị kết quả quý 2)
- Mặc định sort theo Họ và tên: A-Z
- Nếu user lọc Thời gian nhiều quý, các quý cũ nhất hiển thị từ bên trái qua phải
- Hiển thị tối đa 20 items/ trang và có nút chuyển trang 

#   
US-2: \[Web\] Search/Sort/Filter cho bảng xếp hạng Sale theo doanh số/ số lượng GD hàng quý

## Tổng quan:

| **Mục** | **Nội dung** |
| --- | --- |
| US ID | US-02 |
| Tên | \[Web\] Search/Sort/Filter cho bảng xếp hạng Sale theo doanh số/ số lượng GD hàng quý |
| Actor | - Admin hệ thống (11)/ Quản lý đại lý (100): Xem bảng xếp hạng của toàn bộ Sale Vinhomes - Admin đại lý (64): Xem bảng xếp hạng của Sale thuộc Đại lý - Quản lý Vùng chủ quản đại lý: Xem bảng xếp hạng của Sale thuộc các Đại lý quản lý  - Giám đốc miền KD (60): Xem bảng xếp hạng của Sale thuộc miền KD quản lý  - Giám đốc phòng Kinh doanh (23): Xem bảng xếp hạng của Sale thuộc phòng KD quản lý - Trưởng nhóm Kinh doanh (20): Xem bảng xếp hạng của Sale thuộc nhóm KD quản lý - Chuyên viên Kinh doanh (21): Xem bảng xếp hạng của bộ phận sale đang trực thuộc |
| Mô tả | Cho phép user sử dụng: - Thanh tìm kiếm theo Họ tên, ID hệ thống,  Mã nhân viên, Mã định danh của sale - Sort theo Họ tên (A-Z), Số GD, Điểm theo SL GD, Doanh số, Điểm theo doanh số (tăng ↔︎ giảm) - Bộ lọc: Bộ phận kinh doanh; Trạng thái hoạt động; Thời gian; Xếp hạng theo doanh số; Xếp hạng theo số lượng GD |
| Trigger | Truy cập module “**Quản lý sale**” chọn mục “**Bảng xếp hạng sale**” để bắt đầu sử dụng |
| Pre-condition | Được phân quyền và đăng nhập thành công trên hệ thống Agent |
| Post-condition | Hiển thị báo cáo search/sort/filter đang áp dụng |

## ** Mô tả chi tiết:**

| **Hạng mục** | **Tính năng** | **Mô tả** | **Thiết kế** |
| --- | --- | --- | --- |
| Search | Tìm kiếm theo key gần đúng: - Họ và tên - ID hệ thống - Mã nhân viên - Mã định danh |  |  |
| Sort | - Họ và tên theo: A-Z - Số GD, Doanh số, Điểm theo doanh số: tăng-giảm |  |  |
| Filter | Bộ phận kinh doanh | - Bộ lọc bộ phận kinh doanh sale trực thuộc - Radio button (Tất cả, Miền kinh doanh, Phòng kinh doanh, Đội nhóm) kết hợp với chọn nhiều (Multi-select) bằng checkbox bên trong từng nhómMặc định cấp cao nhất mà user được phép truy cập (vd: Admin lọc toàn bộ Miền/Phòng/Nhóm Kinh doanh; Giám đốc Miền KD chỉ lọc được các Phòng/Nhóm trực thuộc miền đó) Check lại bộ lọc có thể ko hợp lý khi áp dụng cho vùng chủ quản |  |
| Dự án | - Multi-choice + search box: Danh sách dự án lấy từ CMS - Khi chọn dự án cụ thể, bảng thống kê chỉ trả về số giao dịch, doanh thu của dự án tương ứng - Các cột còn lại giữ nguyên giá trị Check dev khó thì ko cần làm |  |  |
| Trạng thái hoạt động | Multi-choice, Mặc định Tất cả - Đang hoạt động - Ngưng hoạt động - Chờ xác thực - Nháp |  |  |
| Thời gian | - Multi-choice tối đa 4 quý. Giá trị lựa chọn là các quý theo từng năm. Thứ tụ từ quý hiện tại tới các quý trong quá khứ - Mặc định lọc quý trước đó của quý hiện tại (vd: hiện là 28/9 thuộc quý 3 → mặc định hiển thị kết quả quý 2) |  |  |
| Xếp hạng theo doanh số | Multi-choice, Mặc định Tất cả - Kinh cương - Bạch kim - Vàng - Không xếp hạng |  |  |
|  |  |  |  |

# US-3: \[Web\] Xuất bảng xếp hạng Sale dạng excel về máy

## Tổng quan:

| **Mục** | **Nội dung** |
| --- | --- |
| US ID | US-03 |
| Tên | \[Web\] Search/Sort/Filter cho bảng xếp hạng Sale theo doanh số/ số lượng GD hàng quý |
| Actor | - Admin hệ thống (11)/ Quản lý đại lý (100) - Admin đại lý (64) - Quản lý Vùng chủ quản đại lý - Giám đốc miền KD (60) - Giám đốc phòng Kinh doanh (23) - Trưởng nhóm Kinh doanh (20) Lưu ý: Chỉ có role chuyên viên Kinh doanh (21) không đươc phép xuất báo cáoDev hỏi Quyền xuất có cần role USER\_EXPORTER (25) hiện có? cần check lại quyền này đang phục vụ mục đích gì |
| Mô tả | - Cho phép user sử dụng tính năng xuất báo cáo trên màn hình xếp hạng Sale  - File tải về cần đầy đủ dòng, cột thông tin như danh sách user đang theo dõi trên Agent (Lưu ý xuất full items user đang lọc ra, không chỉ xuất page 1) - Định dạng: excel - Không vượt quá 50.000 dòng (nếu vượt quá tự động cắt danh sách) - Template: [https://vingroupjsc.sharepoint.com/:x:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20Sale%20theo%20doanh%20s%E1%BB%91/Bao\_cao\_xep\_hang\_Sale\_Template.xlsx?d=w683d635278ac4142968cee592439fdec&csf=1&web=1&e=veRtTi](https://vingroupjsc.sharepoint.com/:x:/r/sites/VSF-BDSBL/Shared%20Documents/Ph%E1%BA%A7n%20m%E1%BB%81m%20Kinh%20doanh%20B%C4%90S/Qu%E1%BA%A3n%20l%C3%BD%20s%E1%BA%A3n%20ph%E1%BA%A9m/Internal/1.%20Quan%20ly%20san%20pham/2.%20T%C3%A0i%20li%E1%BB%87u%20d%E1%BB%B1%20%C3%A1n/02.%20D%E1%BB%B1%20%C3%A1n%20Agent/2.%20BA/Qu%E1%BA%A3n%20l%C3%BD%20sale%20%C4%91%E1%BA%A1i%20l%C3%BD/X%E1%BA%BFp%20h%E1%BA%A1ng%20Sale%20theo%20doanh%20s%E1%BB%91/Bao_cao_xep_hang_Sale_Template.xlsx?d=w683d635278ac4142968cee592439fdec&csf=1&web=1&e=veRtTi) |
| Trigger | Trên “**Bảng xếp hạng sale**” bấm nút “xuất báo cáo” |
| Pre-condition | Được phân quyền và đăng nhập thành công trên hệ thống Agent |
| Post-condition | xuất báo cáo thành công về máy |

## **Mô tả thiết kế:**

Từ bảng xếp hạng bấm vào xuất báo cáo:  

  
  
Xác nhận tải báo cáo về:

#   
US-4: \[Web\] Cấu hình công thức cho bảng xếp hạng Sale

## Tổng quan:

| **Mục** | **Nội dung** |
| --- | --- |
| US ID | US-04 |
| Tên | \[Web\] Cấu hình công thức cho bảng xếp hạng Sale |
| Actor | Admin hệ thống (11)/ Quản lý đại lý (100) |
| Mô tả | 1. Cho phép user cấu hình động cho bảng xếp hạng gồm: - Thời gian hiệu lực - Đối tượng áp dụng - Tỷ lệ công thức tính điểm - Định nghĩa các hạng Chỉ được phép cấu hình cho quý đang diễn ra và sắp diễn ra. Các quý đã qua không được phép điều chỉnh cấu hình. 1. Xem lịch sử cấu hình đã lưu Lưu ý: Các quý đã qua user không được phép cấu hình trên UI, dev cấu hình backdated cho quý đã qua |
| Trigger | Trên “**Bảng xếp hạng sale**” bấm vào “**Cấu hình xếp hạng**” |
| Pre-condition | Được phân quyền và đăng nhập thành công trên hệ thống Agent |
| Post-condition | Lưu cấu hình thành công |

## **Mô tả chi tiết:**  

| **Tên trường** | **Định nghĩa** | **Bắt buộc** | **Ví dụ** |
| --- | --- | --- | --- |
| Thời gian hiệu lực | - Chọn các quý áp dụng cấu hình - Chỉ được phép cấu hình cho quý đang diễn ra và sắp diễn ra. Các quý đã qua không được phép điều chỉnh cấu hình. | x | Quý 4-2026 Quý 1-2027 |
| Chọn đối tượng áp dụng | Multi-choice: - Đại lý - Tự doanh - O2O Mỗi đối tượng chỉ thiết lập ở 1 box cấu hình (ví dụ: Đại lý đã chọn ở box 1 thì tới box 2 bị disable, không được chọn) | x | Đại lý |
| Công thức tính điểm | Công thức tính điểm theo doanh số:= a % doanh số quý trước + b % doanh số quý này | x | = 30% doanh số quý trước + 70% doanh số quý này |
| Tên xếp hạng | Mặc định 3 hạng, KHÔNG cho phép cấu hình động: - Kim cương - Bạch kim - Vàng |  |  |
| Điểm doanh số tối thiểu (VNĐ) | Điểm doanh số tối thiểu kết hợp với quota số lượng theo hạng để xác định hạng | x | Kim cương: 80.000.000.000 |
| Quota theo hạng | Quy định quota số lượng sale nhận được hạng theo top từ trên xuống |  | - Kim cương: 100 sale đầu tiên - Bạch kim: 200 sale tiếp theo - Vàng: 300 sale tiếp theo VD: Có 120 sale đạt doanh số từ 80.000.000.000 =\> Chỉ lấy 100 sale có doanh số cao nhất trong 120 sale đó, 20 sale còn lại chuyển xuống hạng Bạch kim TH đặc biệt Check BO nếu 2 sale điểm doanh số bằng nhau cùng tranh chấp hạng 100  =\> Cả 2 sale đều nhận được hạng Kim cương |

## **Mô tả thiết kế:**

Bước 1: Từ màn hình báo cáo bấm vào “Cấu hình xếp hạng”

Bước 2: Tiến hành cấu hình


Bước 3: Xem lại cấu hình đã lưu  
- Cấp 1: mở Modal hiển thị danh sách các cấu hình đã lưu

 Cấp 2 cho phép bấm "Xem chi tiết" để hiển thị thông số của từng cấu hình ở trạng thái chỉ đọc (read-only)


# US-5: \[Web/App\] Gắn tag xếp hạng vinh danh cho Sale  + US-6: \[Web/App\] Tải phiếu chứng nhận xếp hạng cho Sale 

## Tổng quan:

| **Mục** | **Nội dung** |
| --- | --- |
| US ID | US-5 + US-6 |
| Tên | \[Web/App\] Gắn tag xếp hạng vinh danh cho Sale   \[Web/App\] Tải phiếu chứng nhận xếp hạng cho Sale |
| Actor | - Giám đốc miền KD (60) - Giám đốc phòng Kinh doanh (23) - Trưởng nhóm Kinh doanh (20) - Chuyên viên Kinh doanh (21) |
| Mô tả | - Các sale được xếp hạng Kinh cương, Bạch kim, Vàng truy cập web/app Agent xem được huy chương được gắn trên avata cá nhân của chính họ và nút để tải “Phiếu chứng nhận” để tại phiếu xếp hạng về máy - Quý này hiển thị xếp hạng của quý trước và reset khi quý mới bắt đầu - Định dạng file: png/jpeg - Template: Chờ design thiết kế trả vào 5-7/10 |
| Trigger | Truy cập web/app Agent |
| Pre-condition | Được phân quyền và đăng nhập thành công trên hệ thống Agent |
| Post-condition | xem được huy chương được gắn trên avata và tải được Phiếu chứng nhận về máy |

## **Mô tả thiết kế:**

Thiết kế chính thức: Chờ design thiết kế trả vào 5-7/10

### Web: 

Bấm vào avata góc phải màn hình:  
- Gắn badge theo hạng vào avata  
- Nút tải phiếu chứng nhận

### App: 

Gắn badge theo hạng vào avata:  


- Nút tải phiếu chứng nhận
