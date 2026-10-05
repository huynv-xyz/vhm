# vhm-sale-performance — Tài liệu thiết kế chung

Một repository/service/deployment và một PostgreSQL database `sale_performance_db`, gồm module `ranking` và `cycle`. API/worker có thể chạy thành các instance cùng artifact để scale; không tách hai service sở hữu hai DB.

1. [TDD chung](sale-performance-TDD.md): tên, kiến trúc, ownership, integration và implement.
2. [DB và luồng dùng chung](sale-performance-DB-va-luong-du-lieu.md): bảng chuẩn, qualifier theo module, transaction và chống trùng.
3. [9531](../9531/README.md): yêu cầu/thiết kế chi tiết module ranking.
4. [9533](../9533/README.md): yêu cầu/thiết kế chi tiết module cycle.

Schema chung ở đây là tài liệu chuẩn cho bảng dùng chung. Tài liệu từng module giải thích cách sử dụng và bảng nghiệp vụ riêng; không tạo một bản sao hồ sơ/GD/audit/task cho mỗi US.
