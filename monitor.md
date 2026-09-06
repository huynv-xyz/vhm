# Các nhóm cần bổ sung vào dashboard APM Detail Java

## 1. Service Health — Kubernetes

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Khi monitor, chúng ta biết được gì? | Lợi ích mang lại |
|---|---|---|---|---|
| Ready replicas (%) | Available / desired | Tỷ lệ replica đang available so với số replica mong muốn | Service hiện có đủ replica theo cấu hình hay đang chạy thiếu capacity; rollout có hoàn thành hay đang kẹt | Phát hiện service suy giảm trước khi người dùng thấy lỗi; có cơ sở để kiểm tra rollout, readiness, khả năng schedule hoặc scale replica |
| Metric age | Current time − sample time | Số giây kể từ sample HTTP metric gần nhất | Metric còn được cập nhật hay dashboard đang hiển thị dữ liệu cũ/No data do mất telemetry | Tránh kết luận nhầm “không có lỗi” khi Prometheus, exporter hoặc OTel agent đã ngừng gửi metric; giúp kiểm tra độ tin cậy của toàn bộ dashboard |
| Restarts — cả khoảng | Increase restart counter | Tổng số lần container restart trong khoảng thời gian đang chọn | Service có vừa crash/restart hay không, kể cả khi hiện tại đã tự hồi phục | Phát hiện sự cố ngắn đã biến mất khỏi RED metrics; có bằng chứng để kiểm tra log trước khi process chết, probe hoặc node event |
| OOMKilled — cả khoảng | OOM termination reason | Xác định container từng bị dừng do vượt memory limit | Restart có liên quan tới memory/OOM hay không | Rút ngắn thời gian khoanh vùng sự cố; định hướng kiểm tra memory limit, heap, direct buffer, metaspace, native memory và thread stack |
| Pod readiness | Ready = 0/1 | Trạng thái Ready của từng pod | Pod nào đang Ready và pod nào không đủ điều kiện nhận traffic; thời điểm pod mất hoặc lấy lại readiness | Phát hiện pod Running nhưng không phục vụ request; hỗ trợ điều chỉnh readiness probe và kiểm tra dependency lúc startup |
| Request rate theo pod/instance | HTTP rate by instance | Lưu lượng request của từng instance | Traffic có được chia đều giữa các replica hay đang dồn vào một pod; pod nào gần như không nhận request | Phát hiện hot pod, cold pod, stale endpoint, session affinity hoặc load balancing không đều; hỗ trợ quyết định scale và kiểm tra Service endpoints |
| Pod restarts | Restart by pod/container | Chi tiết restart theo từng pod và container theo thời gian | Pod/container cụ thể nào restart, restart lúc nào và có lặp lại theo chu kỳ hay không | Giảm thời gian tìm pod lỗi; phân biệt một pod bất thường với rollout hoặc sự cố ảnh hưởng đồng loạt nhiều pod |
| Termination reason | Last termination reason | Nguyên nhân terminate như `OOMKilled`, `Error`, `ContainerCannotRun` | Lần dừng container liên quan tới OOM, application error hay lỗi khởi chạy container | Định hướng xử lý đúng nhóm nguyên nhân: memory, application log/exit code, image, command, permission, volume hoặc container runtime |
| Phạm vi pod/deployment | Anchored name regex | Giới hạn Kubernetes metric vào workload của service đang chọn | Dữ liệu có thuộc đúng service hay bị lẫn pod có tên tương tự | Tránh kéo nhầm service như `miniportal-vhm-agent-api-*`; đảm bảo replica, restart, OOM và tài nguyên phản ánh đúng workload |

## 2. Container Resources

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Khi monitor, chúng ta biết được gì? | Lợi ích mang lại |
|---|---|---|---|---|
| Container CPU usage theo pod | CPU seconds rate | Số CPU core mà từng pod sử dụng | Mỗi pod đang dùng bao nhiêu CPU; pod nào dùng CPU cao; CPU có tăng tương ứng với traffic hay không | Phát hiện hot pod hoặc workload bất thường; hỗ trợ capacity planning, scale ngang và right-sizing CPU request/limit |
| CPU throttling theo pod (%) | Throttled / total periods | Tỷ lệ chu kỳ CPU bị kernel throttle do chạm CPU limit | Container có việc cần chạy nhưng bị CPU limit chặn hay không; latency tăng có trùng với throttling tăng không | Phân biệt application xử lý chậm với thiếu CPU quota; tránh tăng CPU theo cảm tính và cung cấp bằng chứng để điều chỉnh CPU limit |
| Memory working set theo pod | Working set bytes | Memory pod đang giữ theo góc nhìn cgroup/kubelet | Memory thực tế của pod tăng hay giảm; mức sử dụng có vượt xa Java heap không | Phát hiện memory ngoài heap như direct buffer, metaspace, native allocation, thread stack, OTel agent hoặc sidecar |
| Container memory / limit (%) | Working set / limit | Phần trăm memory đã sử dụng so với limit | Pod còn bao nhiêu headroom trước memory limit; pod nào đang tiến sát ngưỡng OOMKill | Phát hiện sớm nguy cơ OOM; hỗ trợ quyết định tăng limit, giảm heap, tối ưu native memory, sửa leak hoặc scale workload |

## 3. JVM Saturation

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Khi monitor, chúng ta biết được gì? | Lợi ích mang lại |
|---|---|---|---|---|
| Heap utilization (%) theo pod | Heap used / limit | Phần trăm heap đang sử dụng so với giới hạn | Pod JVM nào đang dùng heap cao và còn bao nhiêu headroom | Phát hiện một replica gần cạn heap thay vì bị số liệu của các pod khỏe che lấp; hỗ trợ điều chỉnh heap hoặc scale |
| Old Gen theo pod | Old/Tenured used bytes | Dung lượng object sống lâu trong JVM | Old Gen có giảm sau GC hay đáy Old Gen tăng liên tục theo thời gian | Phát hiện sớm object retention, cache tăng không kiểm soát hoặc memory leak trước khi heap đầy/OOM |
| GC pause p99 theo pod | p99 GC histogram | Thời gian pause p99 của từng loại GC theo instance | HTTP p99 tăng có trùng thời điểm với GC pause dài hay không | Xác định GC có phải nguyên nhân tail latency; hỗ trợ tuning GC, heap và allocation thay vì điều tra sai downstream |
| Allocation rate | Allocated bytes rate | Số byte JVM cấp phát mỗi giây | JVM đang tạo object nhanh tới mức nào; allocation có spike dù TPS không tăng hay không | Phát hiện object churn, serialization hoặc thay đổi code gây GC pressure; cung cấp dữ liệu để quyết định profiling/JFR |
| Thread states | Threads by state | Số thread RUNNABLE, BLOCKED, WAITING hoặc TIMED_WAITING | Thread đang chủ yếu chạy, bị block bởi lock hay chờ dependency/queue | Phát hiện lock contention, nguy cơ deadlock hoặc nhiều thread bị chờ; giúp chọn đúng thời điểm lấy thread dump/JFR |
| Executor active / pool / queue | Active, pool, queued | Mức sử dụng thread pool và số task đang xếp hàng | Executor còn capacity hay active thread đã chạm pool size; queue có đang tăng liên tục không | Phát hiện thread-pool saturation trước khi request timeout hàng loạt; hỗ trợ điều chỉnh pool/queue hoặc xử lý call blocking tới downstream |

## 4. SLO & Error Budget

| Thuộc tính/panel | Metric/công thức | Ý nghĩa | Khi monitor, chúng ta biết được gì? | Lợi ích mang lại |
|---|---|---|---|---|
| Availability SLO | Configured SLO target | Mục tiêu availability dùng để tính allowed error ratio và error budget | Service đang được đánh giá theo mục tiêu 99%, 99.5%, 99.9% hay 99.95% | Chuẩn hóa mục tiêu reliability; tạo cơ sở chung để DevOps, development và product đánh giá chất lượng service |
| Latency SLO | Selected latency bucket | Ngưỡng latency tối đa được coi là đạt mục tiêu | Request phải hoàn thành dưới ngưỡng nào mới được tính là đạt | Cho phép đánh giá latency theo cam kết cụ thể thay vì chỉ nhìn p95/p99 rời rạc |
| Availability SLI | 100 × (1 − 5xx / total) | Tỷ lệ request không gặp lỗi server | Availability thực tế đang là bao nhiêu; service có đạt Availability SLO không | Chuyển error rate kỹ thuật thành chỉ số có thể so trực tiếp với mục tiêu dịch vụ và dùng cho báo cáo reliability |
| Latency SLI | Target bucket / total | Tỷ lệ request hoàn thành dưới ngưỡng latency | Bao nhiêu phần trăm request đạt mục tiêu tốc độ; latency SLO có bị vi phạm không | Đánh giá trực tiếp trải nghiệm theo mục tiêu; hỗ trợ xác định endpoint/service cần tối ưu |
| Error budget remaining — 30d | 1 − actual / allowed errors | Phần trăm ngân sách lỗi còn lại trong chu kỳ 30 ngày | Service đã tiêu bao nhiêu error budget và còn được phép phát sinh bao nhiêu lỗi | Cung cấp tiêu chí khách quan để tiếp tục release, tạm dừng release hoặc ưu tiên reliability work |
| Availability burn rate | Error rate / allowed rate | Tốc độ tiêu error budget so với mức cho phép | Lỗi hiện tại đang tiêu budget nhanh gấp bao nhiêu lần; lỗi là spike ngắn hay incident kéo dài | Phát hiện incident lớn nhanh hơn; kết hợp cửa sổ ngắn/dài để giảm cảnh báo giả và hỗ trợ paging/escalation đúng mức |
