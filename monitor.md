# Các nhóm cần bổ sung vào dashboard APM Detail Java

## 1. Service Health — Kubernetes

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Giá trị giám sát |
|---|---|---|---|
| Ready replicas (%) | Available / desired | Tỷ lệ replica đang available so với số replica mong muốn | Cho biết service có đủ replica hay đang thiếu capacity và rollout có bị kẹt không; giúp phát hiện suy giảm sớm và quyết định kiểm tra readiness, scheduling hoặc scale replica. |
| Metric age | Current time − sample time | Số giây kể từ sample HTTP metric gần nhất | Cho biết telemetry còn cập nhật hay dashboard đang dùng dữ liệu cũ; tránh kết luận nhầm “không có lỗi” khi Prometheus, exporter hoặc OTel agent đã ngừng gửi metric. |
| Restarts — cả khoảng | Increase restart counter | Tổng số lần container restart trong khoảng thời gian đang chọn | Cho biết service có vừa crash/restart dù hiện tại đã hồi phục; giúp phát hiện sự cố ngắn và kiểm tra log trước khi process chết, probe hoặc node event. |
| OOMKilled — cả khoảng | OOM termination reason | Xác định container từng bị dừng do vượt memory limit | Cho biết restart có liên quan tới memory/OOM; giúp khoanh vùng sang memory limit, heap, direct buffer, metaspace, native memory hoặc thread stack. |
| Pod readiness | Ready = 0/1 | Trạng thái Ready của từng pod | Cho biết pod nào đủ điều kiện nhận traffic và thời điểm mất/lấy lại readiness; giúp phát hiện pod Running nhưng không phục vụ request và điều chỉnh readiness probe. |
| Request rate theo pod/instance | HTTP rate by instance | Lưu lượng request của từng instance | Cho biết traffic có phân phối đều hay dồn vào một pod; giúp phát hiện hot pod, cold pod, stale endpoint, session affinity hoặc load balancing không đều. |
| Pod restarts | Restart by pod/container | Chi tiết restart theo từng pod và container theo thời gian | Cho biết pod/container nào restart, thời điểm và mức lặp lại; giúp phân biệt một pod lỗi với rollout hoặc sự cố ảnh hưởng đồng loạt. |
| Termination reason | Last termination reason | Nguyên nhân terminate như `OOMKilled`, `Error`, `ContainerCannotRun` | Cho biết nhóm nguyên nhân dừng container; giúp chuyển đúng hướng xử lý sang memory, application log/exit code, image, command, permission, volume hoặc runtime. |
| Phạm vi pod/deployment | Anchored name regex | Giới hạn Kubernetes metric vào workload của service đang chọn | Cho biết dữ liệu có thuộc đúng workload; tránh lấy nhầm service như `miniportal-vhm-agent-api-*` và bảo đảm replica, restart, OOM, tài nguyên đúng scope. |

## 2. Container Resources

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Giá trị giám sát |
|---|---|---|---|
| Container CPU usage theo pod | CPU seconds rate | Số CPU core mà từng pod sử dụng | Cho biết CPU từng pod và tương quan với traffic; giúp phát hiện hot pod, workload bất thường, lập kế hoạch capacity, scale ngang và right-sizing CPU. |
| CPU throttling theo pod (%) | Throttled / total periods | Tỷ lệ chu kỳ CPU bị kernel throttle do chạm CPU limit | Cho biết container có bị CPU limit chặn và throttling có trùng lúc latency tăng; giúp phân biệt code chậm với thiếu CPU quota và điều chỉnh limit có căn cứ. |
| Memory working set theo pod | Working set bytes | Memory pod đang giữ theo góc nhìn cgroup/kubelet | Cho biết memory thực tế có tăng và có vượt xa Java heap; giúp phát hiện direct buffer, metaspace, native allocation, thread stack, agent hoặc sidecar sử dụng nhiều memory. |
| Container memory / limit (%) | Working set / limit | Phần trăm memory đã sử dụng so với limit | Cho biết pod còn bao nhiêu headroom và pod nào gần OOMKill; giúp quyết định tăng limit, giảm heap, tối ưu native memory, sửa leak hoặc scale workload. |

## 3. JVM Saturation

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Giá trị giám sát |
|---|---|---|---|
| Heap utilization (%) theo pod | Heap used / limit | Phần trăm heap đang sử dụng so với giới hạn | Cho biết pod JVM nào dùng heap cao và headroom còn lại; giúp phát hiện một replica gần cạn heap để điều chỉnh heap hoặc scale. |
| Old Gen theo pod | Old/Tenured used bytes | Dung lượng object sống lâu trong JVM | Cho biết Old Gen có giảm sau GC hay tăng liên tục; giúp phát hiện object retention, cache không kiểm soát hoặc memory leak trước khi heap đầy/OOM. |
| GC pause p99 theo pod | p99 GC histogram | Thời gian pause p99 của từng loại GC theo instance | Cho biết HTTP p99 có tăng cùng GC pause; giúp xác định GC là nguyên nhân tail latency và định hướng tuning GC, heap hoặc allocation. |
| Allocation rate | Allocated bytes rate | Số byte JVM cấp phát mỗi giây | Cho biết tốc độ tạo object và allocation spike dù TPS không tăng; giúp phát hiện object churn, serialization nặng và xác định nhu cầu profiling/JFR. |
| Thread states | Threads by state | Số thread RUNNABLE, BLOCKED, WAITING hoặc TIMED_WAITING | Cho biết thread đang chạy, block hay chờ; giúp phát hiện lock contention, nguy cơ deadlock và chọn thời điểm lấy thread dump/JFR. |
| Executor active / pool / queue | Active, pool, queued | Mức sử dụng thread pool và số task đang xếp hàng | Cho biết executor còn capacity hay queue đang tăng; giúp phát hiện thread-pool saturation trước timeout và điều chỉnh pool/queue hoặc xử lý call blocking. |

## 4. SLO & Error Budget

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Giá trị giám sát |
|---|---|---|---|
| Availability SLO | Configured SLO target | Mục tiêu availability dùng để tính allowed error ratio và error budget | Cho biết service được đánh giá theo mục tiêu 99%, 99.5%, 99.9% hay 99.95%; tạo chuẩn chung để DevOps, development và product đánh giá reliability. |
| Latency SLO | Selected latency bucket | Ngưỡng latency tối đa được coi là đạt mục tiêu | Cho biết request phải hoàn thành dưới ngưỡng nào; giúp đánh giá latency theo cam kết thay vì chỉ nhìn p95/p99 rời rạc. |
| Availability SLI | 100 × (1 − 5xx / total) | Tỷ lệ request không gặp lỗi server | Cho biết availability thực tế và service có đạt SLO; chuyển error rate kỹ thuật thành chỉ số dùng trực tiếp cho báo cáo reliability. |
| Latency SLI | Target bucket / total | Tỷ lệ request hoàn thành dưới ngưỡng latency | Cho biết tỷ lệ request đạt mục tiêu và latency SLO có bị vi phạm; hỗ trợ xác định service/endpoint cần tối ưu. |
| Error budget remaining — 30d | 1 − actual / allowed errors | Phần trăm ngân sách lỗi còn lại trong chu kỳ 30 ngày | Cho biết budget đã tiêu và mức lỗi còn được phép; cung cấp căn cứ để tiếp tục release, dừng release hoặc ưu tiên reliability work. |
| Availability burn rate | Error rate / allowed rate | Tốc độ tiêu error budget so với mức cho phép | Cho biết budget đang cháy nhanh thế nào và lỗi là spike hay incident kéo dài; giúp phát hiện sự cố lớn, giảm cảnh báo giả và escalation đúng mức. |
