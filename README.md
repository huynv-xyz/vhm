# BDSKD-9531 — Bộ tài liệu xếp hạng sale theo doanh số

Cập nhật 05/10/2026. Bộ tài liệu dùng cách trình bày của 9533: phân tích nghiệp vụ trước, xác định nguồn/DB, rồi thiết kế và triển khai từng tính năng. Đây là tài liệu bàn giao, chưa có application code hoặc migration.

## 1. Đọc theo thứ tự

| Tài liệu | Đọc để biết gì? |
| --- | --- |
| [Phân tích PRD](BDSKD-9531-xep-hang-sale-theo-doanh-so-phan-tich-PRD.md) | Mục tiêu, phạm vi và khác biệt PRD/SRS |
| [Phân tích SRS](BDSKD-9531-xep-hang-sale-theo-doanh-so-phan-tich-SRS.md) | Sáu US, công thức, GD hợp lệ, quyền và điểm cần PO xác nhận |
| [Mapping nguồn và DB](BDSKD-9531-xep-hang-sale-theo-doanh-so-doi-chieu-DB.md) | Field profile-mw, contract Kafka và dữ liệu service tự sở hữu |
| [TDD](BDSKD-9531-xep-hang-sale-theo-doanh-so-TDD.md) | Kiến trúc độc lập, schema, engine, API và vận hành |
| [Luồng DB và lộ trình](BDSKD-9531-xep-hang-sale-theo-doanh-so-luong-du-lieu-va-lo-trinh.md) | Từng luồng đọc/ghi, transaction, dữ liệu trước–sau và từng bước implement |
| [Schema profile-mw dùng cho 9531](BDSKD-9531-xep-hang-sale-theo-doanh-so-schema-vhm-profile.md) | Bảng nguồn, quan hệ và cách đọc; liên kết metadata đầy đủ đã khảo sát |

## 2. Nguồn và giới hạn của bản phân tích

- [SRS local](<SRS - Xếp hạng Sale theo doanh số.md>): sáu US, trạng thái Under approve, target 31/10/2026; file không có metadata version trang Confluence.
- [PRD local](PRD_Xep_hang_Sale.odt): bản 0.1 ngày 23/09/2026, chờ phê duyệt; đã đọc nội dung ODT.
- [SRS trên Confluence](https://vin3s.atlassian.net/wiki/spaces/BMAS/pages/3174598094): URL người dùng cung cấp. Ngày 05/10/2026 hai connector Atlassian trả authentication required, web không đọc được trang; chưa đối chiếu bản online với file local.
- Schema profile-mw dùng lại kết quả khảo sát metadata ngày 05/10/2026 trong bộ 9533; không kết nối lại DB hoặc đọc row tài khoản trong đợt này.
- Demo HTML 9531, mẫu chứng nhận, template Excel và Q&A được SRS dẫn link nhưng chưa có nội dung local; chưa kiểm tra UI/template. Không dùng demo 9533 làm UI cho 9531.

## 3. Ranh giới triển khai

Theo định hướng đã chốt ở 9533: độc lập với core-broker, DB nghiệp vụ riêng; hồ sơ/tổ chức đọc các bảng profile-mw bằng datasource chỉ đọc; GD nhận Kafka sale-pipeline. Tên service `vhm-sale-ranking` và PostgreSQL `sale_ranking_db` là đề xuất cho 9531, chưa tạo repo/deployment. Nếu gom 9531/9533 vào cùng một service quản lý sale sau này thì vẫn phải giữ riêng luật GD và kết quả nghiệp vụ.

Không cần đọc bảng, gọi API hoặc import code core-broker. Badge/chứng nhận cung cấp cho web/app qua API của service mới. 9531 không dựa vào kết quả chính thức/thử thách hoặc điều kiện room của 9533 để loại sale.

Thiết kế phân biệt rõ **yêu cầu nguồn**, **đề xuất kỹ thuật** và **luật chờ PO**. Chi tiết chưa chốt không tự được coi là nghiệp vụ đã phê duyệt.
