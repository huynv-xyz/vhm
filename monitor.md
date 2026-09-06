# Các nhóm cần bổ sung vào dashboard APM Detail Java

## 1. Service Health — Kubernetes

Nhóm này bổ sung khả năng theo dõi trạng thái chạy thực tế của service trên Kubernetes. Dashboard APM gốc chỉ nhìn thấy request và JVM của các instance còn phát metric, nên không phát hiện đầy đủ việc thiếu replica, pod mất readiness, restart hoặc OOMKilled.

| Thuộc tính/panel | Metric sử dụng | Ý nghĩa | Thêm vào để monitor được gì? |
|---|---|---|---|
| Ready replicas (%) | `kube_deployment_status_replicas_available` / `kube_deployment_spec_replicas` | Tỷ lệ replica đang available so với số replica mong muốn | Phát hiện service đang chạy thiếu capacity do rollout kẹt, readiness fail, pod chưa schedule được hoặc CrashLoop. |
| Metric age | `time() - timestamp(http_server_request_duration_seconds_count)` | Số giây kể từ sample HTTP metric gần nhất | Phân biệt service thật sự không có traffic với trường hợp OTel agent, exporter hoặc Prometheus scrape đã ngừng cập nhật. |
| Restarts — cả khoảng | `increase(kube_pod_container_status_restarts_total[$__range])` | Tổng số lần container restart trong khoảng thời gian đang chọn | Phát hiện service từng crash rồi tự hồi phục, dù RED metrics hiện tại đã trở lại bình thường. |
| OOMKilled — cả khoảng | `kube_pod_container_status_last_terminated_reason{reason="OOMKilled"}` | Xác định container từng bị dừng do vượt memory limit | Khoanh vùng nhanh restart liên quan tới memory thay vì lỗi application thông thường. |
| Pod readiness | `kube_pod_status_ready{condition="true"}` | Trạng thái Ready 0/1 của từng pod | Phát hiện pod đang Running nhưng không được nhận traffic hoặc bị mất readiness theo thời gian. |
| Request rate theo pod/instance | `rate(http_server_request_duration_seconds_count)` nhóm theo `instance` | Lưu lượng request của từng instance | Phát hiện traffic phân phối không đều, hot pod, cold pod, stale endpoint hoặc một pod không nhận request. |
| Pod restarts | `increase(kube_pod_container_status_restarts_total[$rateinterval])` nhóm theo pod/container | Chi tiết pod và container restart theo thời gian | Xác định chính xác pod nào restart, thời điểm restart và restart có lặp lại hay không. |
| Termination reason | `kube_pod_container_status_last_terminated_reason` | Nguyên nhân terminate như `OOMKilled`, `Error`, `ContainerCannotRun` | Định hướng điều tra sang memory, application crash, image, command, permission hoặc container runtime. |

Selector pod cần giới hạn theo đúng service:

```text
^${service_name:regex}(-deployment)?-.*$
```

Selector Deployment:

```text
^${service_name:regex}(-deployment)?$
```

### Khi monitor nhóm này, chúng ta biết được gì?

- Service hiện có đủ số replica theo cấu hình hay đang chạy thiếu capacity.
- Pod nào đang Ready, pod nào không đủ điều kiện nhận traffic.
- Service có vừa crash/restart hay không, kể cả khi hiện tại đã tự hồi phục.
- Lần dừng container có liên quan tới OOMKilled, application error hay lỗi khởi chạy container.
- Traffic có được chia đều giữa các replica hay đang dồn vào một pod.
- Metric còn đang được cập nhật hay dashboard đang hiển thị dữ liệu cũ/No data do mất telemetry.

### Lợi ích mang lại

- Phát hiện service suy giảm trước khi người dùng thấy lỗi: ví dụ mất một nửa replica nhưng các pod còn lại vẫn đang xử lý được traffic.
- Tránh kết luận nhầm “không có lỗi” khi thực tế Prometheus hoặc OTel agent đã ngừng gửi metric.
- Giảm thời gian khoanh vùng sự cố vì biết ngay pod/container nào có vấn đề và nguyên nhân terminate ban đầu.
- Hỗ trợ kiểm tra rollout: biết deployment mới có đủ replica, pod mới có Ready và có restart sau deploy hay không.
- Phát hiện vấn đề load balancing hoặc Service endpoints khi một pod nhận quá nhiều traffic và pod khác gần như không nhận request.
- Cung cấp bằng chứng để quyết định rollback, restart pod, điều chỉnh probe, scale replica hoặc kiểm tra node/platform.

## 2. Container Resources

Nhóm này bổ sung góc nhìn tài nguyên của toàn bộ container. JVM heap không bao gồm direct buffer, metaspace, native memory, thread stack, OTel agent hoặc sidecar nên không đủ để đánh giá nguy cơ OOM và giới hạn CPU của Kubernetes.

| Thuộc tính/panel | Metric sử dụng | Ý nghĩa | Thêm vào để monitor được gì? |
|---|---|---|---|
| Container CPU usage theo pod | `rate(container_cpu_usage_seconds_total)` | Số CPU core mà từng pod sử dụng | Phát hiện pod dùng CPU cao, hot pod hoặc CPU tăng bất thường so với traffic. |
| CPU throttling theo pod (%) | `rate(container_cpu_cfs_throttled_periods_total)` / `rate(container_cpu_cfs_periods_total)` | Tỷ lệ chu kỳ CPU bị kernel throttle do chạm CPU limit | Xác định latency tăng do CPU limit, kể cả khi CPU usage trung bình chưa nhìn thấy mức cao. |
| Memory working set theo pod | `container_memory_working_set_bytes` | Lượng memory container đang giữ theo góc nhìn cgroup/kubelet | Theo dõi memory thực tế ngoài Java heap và phát hiện native memory hoặc sidecar tăng bất thường. |
| Container memory / limit (%) | `container_memory_working_set_bytes` / `kube_pod_container_resource_limits{resource="memory"}` | Phần trăm memory đã sử dụng so với limit | Xác định pod đang tiến sát memory limit, dự báo nguy cơ OOMKilled và đánh giá nhu cầu tăng limit hoặc sửa memory leak. |

### Khi monitor nhóm này, chúng ta biết được gì?

- Mỗi pod đang dùng bao nhiêu CPU core và pod nào tiêu thụ CPU bất thường.
- Container có bị giới hạn CPU làm chậm quá trình xử lý hay không.
- Memory thực tế của pod đang tăng hay giảm, thay vì chỉ nhìn riêng Java heap.
- Pod còn bao nhiêu khoảng trống trước khi chạm memory limit.
- Latency tăng là do application xử lý chậm hay do container bị CPU throttle.
- OOMKilled có khả năng đến từ heap JVM hay từ direct buffer, metaspace, native memory, thread stack hoặc sidecar.

### Lợi ích mang lại

- Phát hiện sớm nguy cơ OOMKill trước khi container bị kernel dừng.
- Phân biệt lỗi tài nguyên Kubernetes với lỗi logic trong application, giúp giao đúng việc cho DevOps hoặc development team.
- Hỗ trợ right-sizing CPU/memory request và limit dựa trên dữ liệu sử dụng thực tế.
- Tránh tăng CPU hoặc memory theo cảm tính; có bằng chứng pod nào thiếu tài nguyên và thiếu ở mức nào.
- Phát hiện CPU throttling—một nguyên nhân làm p95/p99 tăng nhưng biểu đồ CPU usage trung bình có thể không thể hiện rõ.
- Hỗ trợ capacity planning và quyết định nên scale ngang, tăng limit hay tối ưu application.

## 3. JVM Saturation

Nhóm này bổ sung các tín hiệu cho biết JVM còn capacity hay đang tiến gần giới hạn. Các panel JVM gốc mới chủ yếu cho biết heap, tổng GC time, CPU và tổng thread; chưa thể hiện rõ Old Gen, GC pause, allocation pressure và thread-pool saturation.

| Thuộc tính/panel | Metric sử dụng | Ý nghĩa | Thêm vào để monitor được gì? |
|---|---|---|---|
| Heap utilization (%) theo pod | `jvm_memory_used_bytes` / `jvm_memory_limit_bytes` nhóm theo `instance` | Phần trăm heap đang sử dụng so với giới hạn | Phát hiện pod gần cạn heap và xác định một replica bất thường thay vì nhìn tổng toàn service. |
| Old Gen theo pod | `jvm_memory_used_bytes` với pool `old` hoặc `tenured` | Dung lượng object sống lâu trong JVM | Phát hiện Old Gen tăng liên tục và không giảm sau GC, dấu hiệu object retention hoặc memory leak. |
| GC pause p99 theo pod | `histogram_quantile(0.99, jvm_gc_duration_seconds_bucket)` | Thời gian pause p99 của từng loại GC theo instance | Xác định HTTP p99 tăng có liên quan đến GC pause hoặc stop-the-world hay không. |
| Allocation rate | `rate(jvm_gc_memory_allocated_bytes_total)` | Số byte JVM cấp phát mỗi giây | Phát hiện object churn cao, serialization nặng hoặc thay đổi code làm tăng GC pressure dù heap chưa đầy. |
| Thread states | `jvm_threads_states_threads` nhóm theo `instance`, `state` | Số thread RUNNABLE, BLOCKED, WAITING hoặc TIMED_WAITING | Phát hiện lock contention, nhiều thread bị chờ hoặc workload CPU-bound; hỗ trợ chọn thời điểm lấy thread dump. |
| Executor active / pool / queue | `executor_active_threads`, `executor_pool_size_threads`, `executor_queued_tasks` | Mức sử dụng thread pool và số task đang xếp hàng | Phát hiện executor saturation khi active chạm pool size và queue tiếp tục tăng, nguyên nhân trực tiếp làm request/task phải chờ. |

### Khi monitor nhóm này, chúng ta biết được gì?

- Pod JVM nào đang sử dụng heap cao và còn bao nhiêu headroom.
- Old Gen có được giải phóng sau GC hay tăng liên tục theo thời gian.
- HTTP latency p99 tăng có trùng thời điểm với GC pause dài hay không.
- JVM đang cấp phát object với tốc độ bao nhiêu và có xuất hiện allocation spike không.
- Thread đang chủ yếu RUNNABLE, BLOCKED hay WAITING.
- Executor còn thread trống hay active thread đã chạm pool size.
- Task có bắt đầu xếp hàng và queue có tiếp tục tăng hay không.

### Lợi ích mang lại

- Phát hiện memory leak sớm thông qua xu hướng Old Gen, trước khi heap đầy và pod bị restart/OOMKilled.
- Xác định GC có phải nguyên nhân của tail latency thay vì chỉ thấy HTTP p99 tăng mà không biết lý do.
- Phát hiện object churn cao để development team tập trung profiling allocation, serialization hoặc code vừa thay đổi.
- Phát hiện lock contention hoặc nhiều thread bị chờ để chọn đúng thời điểm lấy thread dump/JFR.
- Nhận biết thread-pool saturation trước khi queue tăng thành timeout hàng loạt.
- Giúp phân biệt ba hướng xử lý: tăng heap/tối ưu memory, tuning GC, hoặc điều chỉnh executor/concurrency.
- Khoanh vùng đúng pod JVM bất thường thay vì số liệu của pod lỗi bị gộp với các pod khỏe.

## 4. SLO & Error Budget

Nhóm này bổ sung khả năng đánh giá reliability theo mục tiêu dịch vụ. RED metrics cho biết đang có bao nhiêu lỗi và latency bao nhiêu, nhưng chưa cho biết mức đó có vi phạm cam kết hay đang tiêu error budget nhanh tới đâu.

| Thuộc tính/panel | Metric/công thức sử dụng | Ý nghĩa | Thêm vào để monitor được gì? |
|---|---|---|---|
| Availability SLO | Biến `$availability_slo`, mặc định `0.999` | Mục tiêu availability dùng làm cơ sở tính error budget | Cho phép đánh giá cùng dữ liệu theo các mục tiêu 99%, 99.5%, 99.9% hoặc 99.95%. |
| Latency SLO | Biến `$slo_bucket` | Ngưỡng latency tối đa được coi là đạt mục tiêu | Cho phép thay đổi ngưỡng latency và tính tỷ lệ request đạt mục tiêu tương ứng. |
| Availability SLI | `100 × (1 - 5xx requests / total requests)` | Tỷ lệ request không gặp lỗi server | Theo dõi availability thực tế theo thời gian và so sánh với Availability SLO. |
| Latency SLI | Request trong bucket `le="$slo_bucket"` / bucket `le="+Inf"` | Tỷ lệ request hoàn thành dưới ngưỡng latency | Biết trực tiếp bao nhiêu phần trăm request đạt mục tiêu latency, thay vì tự suy từ p95/p99. |
| Error budget remaining — 30d | `1 - actual error ratio / allowed error ratio` | Phần trăm ngân sách lỗi còn lại trong chu kỳ 30 ngày | Hỗ trợ quyết định tiếp tục release hay ưu tiên xử lý reliability khi ngân sách lỗi gần hết. |
| Availability burn rate | Error ratio / `(1 - availability_slo)` trên các cửa sổ 5m, 30m, 1h và 6h | Tốc độ tiêu error budget so với mức cho phép | Phát hiện incident đang tiêu budget nhanh; kết hợp cửa sổ ngắn và dài để giảm cảnh báo giả do spike ngắn. |

### Khi monitor nhóm này, chúng ta biết được gì?

- Availability thực tế của service đang là bao nhiêu phần trăm.
- Bao nhiêu phần trăm request đáp ứng mục tiêu latency đã đặt.
- Service hiện có đạt Availability SLO và Latency SLO hay không.
- Trong chu kỳ 30 ngày còn bao nhiêu error budget.
- Lỗi hiện tại đang tiêu ngân sách nhanh gấp bao nhiêu lần mức cho phép.
- Lỗi chỉ là spike ngắn hay đã kéo dài đủ lâu để ảnh hưởng nghiêm trọng tới SLO.

### Lợi ích mang lại

- Chuyển từ cách monitor “có lỗi hay không” sang “mức lỗi này có vi phạm cam kết dịch vụ hay không”.
- Cung cấp tiêu chí khách quan để quyết định incident có cần paging/escalation ngay hay chỉ cần theo dõi.
- Hỗ trợ quyết định tiếp tục release, tạm dừng release hoặc ưu tiên xử lý reliability debt dựa trên error budget còn lại.
- Multi-window burn rate giúp bắt incident lớn nhanh hơn nhưng hạn chế cảnh báo giả do một spike rất ngắn.
- Tạo ngôn ngữ chung giữa DevOps, development và product: availability, latency compliance và ngân sách lỗi.
- Hỗ trợ báo cáo chất lượng dịch vụ theo mục tiêu thay vì chỉ gửi các biểu đồ kỹ thuật rời rạc.
